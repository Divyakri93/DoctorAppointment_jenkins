pipeline {
    agent any

    environment {
        DOCKER_IMAGE_BACKEND  = "doctor-appointment-backend"
        DOCKER_IMAGE_FRONTEND = "doctor-appointment-frontend"
        DOCKER_IMAGE_ADMIN    = "doctor-appointment-admin"
        DOCKER_TAG            = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Cloning repository...'
                checkout scm
            }
        }

        stage('Create Backend .env') {
            steps {
                withCredentials([
                    string(credentialsId: 'MONGODB_URI',           variable: 'MONGODB_URI'),
                    string(credentialsId: 'CLOUDINARY_NAME',       variable: 'CLOUDINARY_NAME'),
                    string(credentialsId: 'CLOUDINARY_API_KEY',    variable: 'CLOUDINARY_API_KEY'),
                    string(credentialsId: 'CLOUDINARY_SECRET_KEY', variable: 'CLOUDINARY_SECRET_KEY'),
                    string(credentialsId: 'JWT_SECRET',            variable: 'JWT_SECRET'),
                    string(credentialsId: 'RAZORPAY_KEY_ID',       variable: 'RAZORPAY_KEY_ID'),
                    string(credentialsId: 'RAZORPAY_KEY_SECRET',   variable: 'RAZORPAY_KEY_SECRET')
                ]) {
                    powershell '''
                        $lines = @(
                            "MONGODB_URI=$env:MONGODB_URI",
                            "CLOUDINARY_NAME=$env:CLOUDINARY_NAME",
                            "CLOUDINARY_API_KEY=$env:CLOUDINARY_API_KEY",
                            "CLOUDINARY_SECRET_KEY=$env:CLOUDINARY_SECRET_KEY",
                            "ADMIN_EMAIL=admin@prescripto.com",
                            "ADMIN_PASSWORD=qwerty123",
                            "JWT_SECRET=$env:JWT_SECRET",
                            "RAZORPAY_KEY_ID=$env:RAZORPAY_KEY_ID",
                            "RAZORPAY_KEY_SECRET=$env:RAZORPAY_KEY_SECRET",
                            "CURRENCY=INR"
                        )
                        $lines | Set-Content -Path "backend\\.env" -Encoding UTF8
                        Write-Host ".env file created successfully for backend"
                    '''
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                echo 'Building Docker images (npm install + build handled inside Docker)...'
                bat """
                    docker build -t %DOCKER_IMAGE_BACKEND%:%DOCKER_TAG%  ./backend
                    docker build -t %DOCKER_IMAGE_FRONTEND%:%DOCKER_TAG% ./frontend
                    docker build -t %DOCKER_IMAGE_ADMIN%:%DOCKER_TAG%    ./admin
                """
            }
        }

        stage('Deploy with Docker Compose') {
            steps {
                echo 'Stopping old containers...'
                bat 'docker-compose down --remove-orphans'
                echo 'Starting new containers...'
                bat 'docker-compose up -d --build'
            }
        }

        stage('Health Check') {
            steps {
                echo 'Waiting 30 seconds for services to start...'
                sleep(time: 30, unit: 'SECONDS')
                bat '''
                    curl -s -o nul -w "Backend  (port 5000): HTTP %%{http_code}\n" http://localhost:5000 || echo Backend is starting up...
                    curl -s -o nul -w "Frontend (port 5173): HTTP %%{http_code}\n" http://localhost:5173 || echo Frontend is starting up...
                    curl -s -o nul -w "Admin    (port 5174): HTTP %%{http_code}\n" http://localhost:5174 || echo Admin is starting up...
                    exit /b 0
                '''
            }
        }
    }

    post {
        success {
            echo '========================================='
            echo '✅ Deployment successful! Containers are running.'
            echo 'Backend  → http://localhost:5000'
            echo 'Frontend → http://localhost:5173'
            echo 'Admin    → http://localhost:5174'
            echo '========================================='
        }
        failure {
            echo '❌ Pipeline failed at a build/config stage.'
            echo 'Note: If containers are running, they will NOT be stopped.'
            echo 'Run: docker-compose down   (only if you want to stop them)'
        }
        always {
            powershell 'Remove-Item -Path "backend\\.env" -ErrorAction SilentlyContinue'
            echo 'Pipeline finished. Secrets cleaned up.'
        }
    }
}
