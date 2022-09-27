#!/usr/bin/env groovy
pipeline {

    agent any
/*
        docker {
                reuseNode true
                image 'maven:3.5.4-alpine'
                args '-v /root/.m2:/root/.m2'
        }
 */
    environment {
        ECR_PROTOCOL = 'https://'
        ECR_URL = 'registry.gitlab.com'
        ECR_CREDENTIAL = 'gitlab_user_pass'
        REPO_NAME = 'sarwandria/spring-boot'
        GET_COMMIT_HASH = get_commit_hash()
        DOCKERFILE = dockerfilename()
    }
    stages {
        stage('Checkout config') {
            steps {
                sh 'mkdir -p config'
                dir('config') {
                    git branch: 'main',
                        url: 'https://github.com/sarwandria/IAC.git'
                }
            }
        }
        stage('Copy Config to development') {
            when {
                expression {
                    return ((env.GIT_BRANCH == 'origin/develop'))
                }
            }
            steps {
                sh '''
        cp config/app/spring-boot/development/* .
        rm -rf config
        '''
            }
        }
        stage('Copy Config to production') {
            when {
                expression {
                    return ((env.GIT_BRANCH =~ /origin\/release.*$/))
                }
            }
            steps {
                sh '''
                    cp config/app/spring-boot/production/* .
                    rm -rf config
                    '''
            }
        }
        stage('Build Package') {
            agent {
                docker {
                    /*
                     * Reuse the workspace on the agent defined at top-level of Pipeline but run inside a container.
                     * In this case we are running a container with maven so we don't have to install specific versions
                     * of maven directly on the agent
                     */
                    reuseNode true
                    image 'maven:3.5.4-alpine'
                    args '-v /root/.m2:/root/.m2'
                }
            }
            stages('Build all packages') {
                stage('Build App') {
                    steps {
                        sh 'mvn -B -DskipTests clean package'
                    }
                }
                stage('test') {
                    steps {
                        sh 'mvn test'
                    }
                    post {
                        always {
                            junit 'target/surefire-reports/*.xml'
                        }
                    }
                }
            }
        }
        stage('Docker Image Build Development') {
            when {
                expression {
                    return ((env.GIT_BRANCH == 'origin/develop'))
                }
            }
            steps {
                script {
                    docker.build("${ECR_URL}/${REPO_NAME}:dev-${GET_COMMIT_HASH}", "-f ${DOCKERFILE} .")
                    docker.withRegistry(ECR_PROTOCOL+ECR_URL,ECR_CREDENTIAL) {
                        docker.image("${ECR_URL}/${REPO_NAME}:dev-${GET_COMMIT_HASH}").push("dev-${GET_COMMIT_HASH}")
                    }
                }
            }
        }
        stage('Docker Image Build Production') {
            when {
                expression {
                    return ((env.GIT_BRANCH =~ /origin\/release.*$/))
                }
            }
            steps {
                script {
                     docker.build("${ECR_URL}/${REPO_NAME}:prod-${GET_COMMIT_HASH}", "-f ${DOCKERFILE} .")
                     docker.withRegistry(ECR_PROTOCOL+ECR_URL,ECR_CREDENTIAL) {
                        docker.image("${ECR_URL}/${REPO_NAME}:prod-${GET_COMMIT_HASH}").push("prod-${GET_COMMIT_HASH}")
                    }
                }
            }
        }
        stage('Deploy to development') {
            when {
                expression {
                    return ((env.GIT_BRANCH == 'origin/develop') )
                }
            }
            steps {
                sh "docker -H tcp://18.138.22.148:2377 stack deploy spring-boot-dev -c docker-compose.yml --with-registry-auth --resolve-image=always"
            }
        }
        stage('Deploy to production') {
            when {
                expression {
                    return ((env.GIT_BRANCH =~ /origin\/release.*$/) || (env.GIT_BRANCH =~ /origin\/infra-dev.*$/))
                }
            }
            steps {
                sh "export DOCKER_TLS_VERIFY= && docker -H tcp://18.138.22.148:2375 stack deploy spring-boot-prod -c docker-compose.yml --with-registry-auth --resolve-image=always"
            }
        }
    }
}
/*
# Return a short commit (from 0 to 7)
*/
def get_commit_hash() {
    return sh(script: "git rev-parse HEAD | cut -c1-7", returnStdout: true).trim()
}
def dockerfilename() {
    String name=''
    if ((env.GIT_BRANCH =~ /origin\/release.*$/)) {
        name = 'Dockerfile.production'
    } else {
        name = 'Dockerfile.development'
    }
    return name
}