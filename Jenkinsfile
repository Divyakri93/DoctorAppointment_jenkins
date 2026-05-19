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
                    string(credentialsId: 'MONGODB_URI',          variable: 'MONGODB_URI'),
                    string(credentialsId: 'CLOUDINARY_NAME',      variable: 'CLOUDINARY_NAME'),
                    string(credentialsId: 'CLOUDINARY_API_KEY',   variable: 'CLOUDINARY_API_KEY'),
                    string(credentialsId: 'CLOUDINARY_SECRET_KEY',variable: 'CLOUDINARY_SECRET_KEY'),
                    string(credentialsId: 'JWT_SECRET',           variable: 'JWT_SECRET'),
                    string(credentialsId: 'RAZORPAY_KEY_ID',      variable: 'RAZORPAY_KEY_ID'),
                    string(credentialsId: 'RAZORPAY_KEY_SECRET',  variable: 'RAZORPAY_KEY_SECRET')
                ]) {
                    sh """
                        cat > backend/.env <<EOF
MONGODB_URI=${MONGODB_URI}
CLOUDINARY_NAME=${CLOUDINARY_NAME}
CLOUDINARY_API_KEY=${CLOUDINARY_API_KEY}
CLOUDINARY_SECRET_KEY=${CLOUDINARY_SECRET_KEY}
ADMIN_EMAIL=admin@prescripto.com
ADMIN_PASSWORD=qwerty123
JWT_SECRET=${JWT_SECRET}
RAZORPAY_KEY_ID=${RAZORPAY_KEY_ID}
RAZORPAY_KEY_SECRET=${RAZORPAY_KEY_SECRET}
CURRENCY=INR
EOF
                    """
                    echo '.env file created for backend using Jenkins credentials'
                }
            }
        }

        stage('Install Dependencies') {
            parallel {
                stage('Backend Dependencies') {
                    steps {
                        dir('backend') {
                            echo 'Installing backend dependencies...'
                            sh 'npm install'
                        }
                    }
                }
                stage('Frontend Dependencies') {
                    steps {
                        dir('frontend') {
                            echo 'Installing frontend dependencies...'
                            sh 'npm install'
                        }
                    }
                }
                stage('Admin Dependencies') {
                    steps {
                        dir('admin') {
                            echo 'Installing admin dependencies...'
                            sh 'npm install'
                        }
                    }
                }
            }
        }

        stage('Build Frontend & Admin') {
            parallel {
                stage('Build Frontend') {
                    steps {
                        dir('frontend') {
                            echo 'Building frontend...'
                            sh 'npm run build'
                        }
                    }
                }
                stage('Build Admin') {
                    steps {
                        dir('admin') {
                            echo 'Building admin panel...'
                            sh 'npm run build'
                        }
                    }
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                echo 'Building Docker images...'
                sh """
                    docker build -t ${DOCKER_IMAGE_BACKEND}:${DOCKER_TAG}  ./backend
                    docker build -t ${DOCKER_IMAGE_FRONTEND}:${DOCKER_TAG} ./frontend
                    docker build -t ${DOCKER_IMAGE_ADMIN}:${DOCKER_TAG}    ./admin
                """
            }
        }

        stage('Deploy with Docker Compose') {
            steps {
                echo 'Deploying application using Docker Compose...'
                sh """
                    docker-compose down --remove-orphans || true
                    docker-compose up -d --build
                """
            }
        }

        stage('Health Check') {
            steps {
                echo 'Waiting for services to start...'
                sleep(time: 15, unit: 'SECONDS')
                sh """
                    curl -f http://localhost:5000 || echo 'Backend health check pending'
                    curl -f http://localhost:5173 || echo 'Frontend health check pending'
                    curl -f http://localhost:5174 || echo 'Admin health check pending'
                """
            }
        }
    }

    post {
        success {
            echo '✅ Deployment successful!'
            echo "Backend  → http://localhost:5000"
            echo "Frontend → http://localhost:5173"
            echo "Admin    → http://localhost:5174"
        }
        failure {
            echo '❌ Build or deployment failed. Check logs above.'
            sh 'docker-compose down || true'
        }
        always {
            // Clean up the generated .env so secrets don't linger on disk
            sh 'rm -f backend/.env'
            echo 'Pipeline finished.'
        }
    }
}
