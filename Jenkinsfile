pipeline { 
    agent any
    
    tools {
        jdk 'Java21'
        maven 'Maven3'
        nodejs 'Node18'
    }

    environment {
        MYSQL_DB = 'Portfolio'
        MYSQL_PORT = '3306'
        MYSQL_ROOT_PASSWORD = 'test_pass'
        MYSQL_USER = 'test'
        MYSQL_PASSWORD = 'pass'
        APP_NAME = "portfolio-app-cicd-pipeline"
        RELEASE = "1.0.0"
        DOCKER_AUTH = credentials('dockerhub')
        IMAGE_NAME = "${DOCKER_AUTH_USR}/${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
        JENKINS_API_TOKEN = credentials("JENKINS_API_TOKEN")
    }

    stages {
      stage('Start MySQL (with password)') {
      steps {
        sh 'docker rm -f mysql-test || true'
        sh """
          docker run --name mysql-test \
            -e MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD} \
            -e MYSQL_DATABASE=${MYSQL_DB} \
            -e MYSQL_USER=${MYSQL_USER} \
            -e MYSQL_PASSWORD=${MYSQL_PASSWORD} \
            -p ${MYSQL_PORT}:3306 \
            -d mysql:8.0 \
            --default-authentication-plugin=mysql_native_password

          echo "Waiting for MySQL to be ready..."
          for i in {1..30}; do
            docker exec mysql-test mysqladmin ping -h 127.0.0.1 -uroot -p${MYSQL_ROOT_PASSWORD} --silent && break
            sleep 2
          done
        """
      }
    }

        stage("Cleanup Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Checkout from SCM") {
            steps {
                git branch: 'main', credentialsId: 'github', url: 'https://github.com/mira33ch/prototype'
            }
        }

        stage("Build Application") {
            steps {
                script {
                    // Build Angular frontend
                    dir('portfolio-frontend') {
                        sh 'npm install'
                        sh 'npm run build --prod'
                    }

                    // Build Spring Boot backend
                    dir('demo1') {
                        sh 'mvn clean package -DskipTests'
                    }
                }
            }
        }

       stage("Test Application") {
  steps {
    dir('demo1') {
      sh """
        mvn -B test \
          -Dspring.datasource.url=jdbc:mysql://127.0.0.1:3306/${MYSQL_DB}?createDatabaseIfNotExist=true\\&useSSL=false\\&allowPublicKeyRetrieval=true \
          -Dspring.datasource.username=root \
          -Dspring.datasource.password=${MYSQL_ROOT_PASSWORD}
      """
    }
  }
}


    stage('SonarQube Analysis Backend') {
            steps {
                dir('demo1') {
                    script {
                        withSonarQubeEnv(credentialsId: 'jenkins-sonarqube-token') {
                            sh 'mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar'
                        }
                    }
                }
            }
        }

        stage('SonarQube Frontend Analysis') {
            steps {
                dir('portfolio-frontend') {
                    script {
                        withSonarQubeEnv(credentialsId: 'jenkins-sonarqube-token') {
                            sh 'npm install sonar-scanner'
                            sh 'npx sonar-scanner -Dsonar.projectKey=frontend-project-key -Dsonar.sources=src'
                        }
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'jenkins-sonarqube-token'
                }
            }
        }

        stage('Build & Push Docker Images') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub') {
                        def backendImage = docker.build("${IMAGE_NAME}-backend:${IMAGE_TAG}", 'demo1/')
                        backendImage.push()
                        backendImage.push('latest')

                        def frontendImage = docker.build("${IMAGE_NAME}-frontend:${IMAGE_TAG}", 'portfolio-frontend/')
                        frontendImage.push()
                        frontendImage.push('latest')
                    }
                }
            }
        }

        stage("Trivy Scan") {
            steps {
                script {
                    sh 'docker run -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image mariem360/portfolio-app-cicd-pipeline-frontend:latest --no-progress --scanners vuln --exit-code 0 --severity HIGH,CRITICAL --format table'
                    sh 'docker run -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image mariem360/portfolio-app-cicd-pipeline-backend:latest --no-progress --scanners vuln --exit-code 0 --severity HIGH,CRITICAL --format table'
                }
            }
        }

        stage('Stop MySQL') {
            steps {
                sh '''
                    docker stop mysql-test || true
                    docker rm mysql-test || true
                '''
            }
        }

        stage('Cleanup Artifacts') {
            steps {
                script {
                    sh "docker rmi ${IMAGE_NAME}-backend:${IMAGE_TAG} || true"
                    sh "docker rmi ${IMAGE_NAME}-backend:latest || true"
                    sh "docker rmi ${IMAGE_NAME}-frontend:${IMAGE_TAG} || true"
                    sh "docker rmi ${IMAGE_NAME}-frontend:latest || true"
                }
            }
        }
          stage('Trigger CD Pipeline') {
            steps {
                script {
                    sh "curl -v -k --user clouduser:${JENKINS_API_TOKEN} -X POST -H 'cache-control: no-cache' -H 'content-type: application/x-www-form-urlencoded' --data 'IMAGE_TAG=${IMAGE_TAG}' 'localhost:8080/job/gitops-app-cd/buildWithParameters?token=gitops-token'"
                }
            }
        }
    }
   
    

    post {
        always {
            echo 'Pipeline finished'
        }
    }
}
