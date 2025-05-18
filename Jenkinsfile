pipeline {
    agent any

    triggers {
        githubPush()
    }

    environment {
        // Credentials will be accessed securely
        MONGO_URI_CRED = credentials('mongo-uri')
        MONGO_PASS_CRED = credentials('mongo-pass')
        
        // Docker registry configuration
        DOCKER_REGISTRY = "4mirul"
        CLIENT_IMAGE = "minido-client"
        SERVER_IMAGE = "minido-server"
        DOCKER_TAG = "${env.BUILD_NUMBER}"  // Simplified tag - just build number
    }

    stages {
        stage('Checkout') {
            when { 
                branch 'main'
            }
            steps {
                git branch: 'main',
                     credentialsId: 'jenkins-github',
                     url: 'git@github.com:4mirul/minido.git'
            }
        }

        stage('Build Images') {
            when { 
                branch 'main'
            }
            parallel {
                stage('Build Client') {
                    steps {
                        dir('app/client') {
                            script {
                                docker.build("${DOCKER_REGISTRY}/${CLIENT_IMAGE}:${DOCKER_TAG}", ".")
                            }
                        }
                    }
                }
                stage('Build Server') {
                    steps {
                        dir('app/server') {
                            script {
                                docker.build("${DOCKER_REGISTRY}/${SERVER_IMAGE}:${DOCKER_TAG}", ".")
                            }
                        }
                    }
                }
            }
        }

        stage('Push to Registry') {
            when { 
                branch 'main'
            }
            steps {
                script {
                    // Push client image with build number tag and latest
                    docker.withRegistry("https://registry.hub.docker.com", 'jenkins-dockerhub') {
                        docker.image("${DOCKER_REGISTRY}/${CLIENT_IMAGE}:${DOCKER_TAG}").push()
                        docker.image("${DOCKER_REGISTRY}/${CLIENT_IMAGE}:${DOCKER_TAG}").push('latest')
                    }
                    
                    // Push server image with build number tag and latest
                    docker.withRegistry("https://registry.hub.docker.com", 'jenkins-dockerhub') {
                        docker.image("${DOCKER_REGISTRY}/${SERVER_IMAGE}:${DOCKER_TAG}").push()
                        docker.image("${DOCKER_REGISTRY}/${SERVER_IMAGE}:${DOCKER_TAG}").push('latest')
                    }
                }
            }
        }

        stage('Prepare Deployment') {
            steps {
                sh """
                    docker tag ${DOCKER_REGISTRY}/${CLIENT_IMAGE}:${DOCKER_TAG} minido-client:latest
                    docker tag ${DOCKER_REGISTRY}/${SERVER_IMAGE}:${DOCKER_TAG} minido-server:latest
                """
                sh 'docker volume create minido_mongodb_data || true'
            }
        }

        stage('Stop and Remove Existing Containers') {
            steps {
                sh '''
                    docker stop minido-mongodb minido-server minido-client || true
                    docker rm minido-mongodb minido-server minido-client || true
                '''
            }
        }

        stage('Deploy') {
            steps {
                // Start MongoDB
                script {
                    withCredentials([
                        string(credentialsId: 'mongo-pass', variable: 'MONGO_PASS')
                    ]) {
                        sh '''
                            docker run -d \
                                --name minido-mongodb \
                                -e PUID=1000 \
                                -e PGID=1000 \
                                -e TZ=Asia/Kuala_Lumpur \
                                -e MONGO_INITDB_ROOT_USERNAME=root \
                                -e MONGO_INITDB_ROOT_PASSWORD=$MONGO_PASS \
                                -p 27018:27017 \
                                -v minido_mongodb_data:/data/db \
                                --restart unless-stopped \
                                mongo:8.0.9
                        '''
                    }
                }

                // Start Server
                script {
                    withCredentials([
                        string(credentialsId: 'mongo-uri', variable: 'MONGO_URI')
                    ]) {
                        sh '''
                            docker run -d \
                                --name minido-server \
                                -e PUID=1000 \
                                -e PGID=1000 \
                                -e TZ=Asia/Kuala_Lumpur \
                                -e NODE_ENV=production \
                                -e MONGO_URI=$MONGO_URI \
                                -e PORT=30000 \
                                -p 30000:30000 \
                                --restart unless-stopped \
                                --link minido-mongodb:mongodb \
                                ${DOCKER_REGISTRY}/${SERVER_IMAGE}:latest
                        '''
                    }
                }

                // Start Client
                sh """
                    docker run -d \
                        --name minido-client \
                        -e PUID=1000 \
                        -e PGID=1000 \
                        -e TZ=Asia/Kuala_Lumpur \
                        -p 8380:80 \
                        --restart unless-stopped \
                        --link minido-server:minido-server \
                        ${DOCKER_REGISTRY}/${CLIENT_IMAGE}:latest
                """
            }
        }

        stage('Verify') {
            steps {
                retry(3) {
                    sh 'curl --retry 3 --fail http://localhost:8380 || true'
                    sh 'curl --retry 3 --fail http://localhost:30000 || true'
                }
            }
        }
    }

    post {
        always {
            echo "Cleaning up workspace and Docker resources"
            cleanWs()
            sh '''
                docker container prune -f || true
                docker image prune -f --filter "until=24h" || true
            '''
        }
        success {
            echo """
            ============ DEPLOYMENT SUCCESS ============
            Build: #${env.BUILD_NUMBER}
            Images:
            - ${DOCKER_REGISTRY}/${CLIENT_IMAGE}:${DOCKER_TAG}
            - ${DOCKER_REGISTRY}/${SERVER_IMAGE}:${DOCKER_TAG}
            - ${DOCKER_REGISTRY}/${CLIENT_IMAGE}:latest
            - ${DOCKER_REGISTRY}/${SERVER_IMAGE}:latest
            ===========================================
            """
        }
        failure {
            archiveArtifacts artifacts: '**/docker*.log,**/npm*.log', allowEmptyArchive: true
            echo """
            ============= DEPLOYMENT FAILED ============
            Build: #${env.BUILD_NUMBER}
            Logs: ${env.BUILD_URL}console
            ===========================================
            """
        }
    }
}