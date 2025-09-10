pipeline {
    agent any

    environment {
        REGISTRY = "mehdi621docker"   
    }

    stages {
        stage("Git Clone") {
            steps {
                git url: "https://github.com/mehdi621-git/chattingo.git", branch: "main"
            }
        }

        stage("Image Build") {
            steps {
                sh "docker compose build frontend"
                sh "docker compose build backend"
            }
        }
       stage('Filesystem Scan') {
            steps {
                sh 'trivy fs --exit-code 0 --severity HIGH,CRITICAL .' 
            }
        }
       stage('Image Scan') {
            steps {
                sh """
                  trivy image --exit-code 0 --severity HIGH,CRITICAL $REGISTRY/chattingofrontend:latest
                  trivy image --exit-code 0 --severity HIGH,CRITICAL $REGISTRY/chattingobackend:latest
                """
            }
        }

        stage("Pushing to Registry") {
            steps {
                withCredentials([usernamePassword(credentialsId: "hack", usernameVariable: "DOCKER_USER", passwordVariable: "DOCKER_PASS")]) {
                    sh "echo ${DOCKER_PASS} | docker login -u ${DOCKER_USER} -p ${DOCKER_PASS}"
                    sh "docker push ${REGISTRY}/chattingofrontend:latest"
                    sh "docker push ${REGISTRY}/chattingobackend:latest"
                }
            }
        }

        stage("Update Compose") {
            steps {
                sh """
                  sed -i 's|image: .*/chattingofrontend:.*|image: ${REGISTRY}/chattingofrontend:latest|' docker-compose.yml
                  sed -i 's|image: .*/chattingobackend:.*|image: ${REGISTRY}/chattingobackend:latest|' docker-compose.yml
                """
            }
        }

        stage("Deploy") {
            steps {
                
                sh "docker compose up -d"
            }
        }
      stage('Certbot SSL') {
          steps {
            sh '''
        docker run --rm \
          -v $(pwd)/letsencrypt:/etc/letsencrypt \
          -v $(pwd)/letsencrypt-www:/var/www/certbot \
          certbot/certbot certonly --webroot \
          --webroot-path=/var/www/certbot \
          -d chattingo.duckdns.org --agree-tos --email mm3328393@gmail.com --non-interactive
        '''
    }
}
      stage('Restart Frontend') {
             steps {
        sh 'docker compose restart frontend'
    }
}


    }
}
