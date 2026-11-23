pipeline {
    agent any

    tools {
        maven 'MAVEN_HOME'   // Manage Jenkins > Tools: Maven installation named "Maven-3"
        jdk   'JDK17'    // Manage Jenkins > Tools: JDK installation named "JDK-11"
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        IMAGE_NAME     = 'e-ticketing'
        CONTAINER_NAME = 'e-ticketing-app'
        HOST_PORT      = '8001'
        // Optional: set to push to a registry (e.g. 'docker.io/youruser')
        // REGISTRY    = 'docker.io/youruser'
        // REGISTRY_CREDS = 'dockerhub-creds'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                // "clean" matters: target/ may hold a stale ots.war, and the
                // Dockerfile copies target/*.war
                sh 'mvn -B clean package -DskipTests'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn -B test'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Archive') {
            steps {
                archiveArtifacts artifacts: 'target/*.war', fingerprint: true
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} -t ${IMAGE_NAME}:latest .'
            }
        }

        // Uncomment together with REGISTRY / REGISTRY_CREDS above
        // stage('Docker Push') {
        //     steps {
        //         withCredentials([usernamePassword(credentialsId: env.REGISTRY_CREDS,
        //                 usernameVariable: 'REG_USER', passwordVariable: 'REG_PASS')]) {
        //             sh '''
        //                 echo "$REG_PASS" | docker login -u "$REG_USER" --password-stdin
        //                 docker tag ${IMAGE_NAME}:${BUILD_NUMBER} ${REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER}
        //                 docker push ${REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER}
        //             '''
        //         }
        //     }
        // }

        stage('Deploy') {
            when { branch 'main' }
            steps {
                sh '''
                    docker rm -f ${CONTAINER_NAME} || true
                    docker run -d --name ${CONTAINER_NAME} \
                        --restart unless-stopped \
                        -p ${HOST_PORT}:8080 \
                        ${IMAGE_NAME}:${BUILD_NUMBER}
                '''
            }
        }
    }

    post {
        success {
            echo "Build ${env.BUILD_NUMBER} succeeded."
        }
        failure {
            echo "Build ${env.BUILD_NUMBER} failed."
        }
        always {
            sh 'docker image prune -f || true'
            cleanWs()
        }
    }
}
