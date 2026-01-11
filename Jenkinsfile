pipeline {
  agent any

  environment {
    COMPOSE = 'docker compose -f docker-compose.yml -f docker-compose.ci.yml'
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Pre-clean (stop old containers)') {
      steps {
        bat '''
        echo === Pre-clean ===
        %COMPOSE% down -v || exit /b 0
        '''
      }
    }

    stage('Docker Build') {
      steps {
        bat '''
        echo === Docker Build ===
        %COMPOSE% build
        '''
      }
    }

    stage('Start Containers (DB only)') {
      steps {
        bat '''
        echo === Start DB ===
        %COMPOSE% up -d db
        %COMPOSE% ps
        '''
      }
    }

    stage('Wait DB Ready') {
      steps {
        bat '''
        echo === Wait DB Ready ===

        for /L %%i in (1,1,20) do (
          %COMPOSE% exec -T db mysqladmin ping -uroot -proot --silent && (
            echo MySQL ready
            exit /b 0
          )
          echo Waiting MySQL... %%i
          ping 127.0.0.1 -n 3 >nul
        )

        echo ERROR: MySQL not ready after retries
        %COMPOSE% logs db
        exit /b 1
        '''
      }
    }

    stage('Laravel Setup') {
      steps {
        bat '''
        echo === Laravel Setup ===

        rem Jalankan artisan via "run --rm" supaya gak butuh app container running
        %COMPOSE% run --rm app sh -lc "cp -n .env.example .env || true && php artisan key:generate --force"
        %COMPOSE% run --rm app php artisan migrate --force
        %COMPOSE% run --rm app php artisan config:clear
        %COMPOSE% run --rm app php artisan cache:clear
        '''
      }
    }

    stage('Smoke Test') {
      steps {
        bat '''
        echo === Smoke Test ===
        %COMPOSE% run --rm app php artisan route:list
        '''
      }
    }
  }

  post {
    always {
      bat '''
      echo === Final docker ps ===
      %COMPOSE% ps
      '''
    }
    cleanup {
      bat '''
      echo === Cleanup ===
      %COMPOSE% down -v
      '''
    }
  }
}
