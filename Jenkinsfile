pipeline {
    agent any
    environment {

        REACT_APP_VERSION = "1.0.$BUILD_ID"
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
        NETLIFY_SITE_ID = '6f30e67b-90a6-40ee-bc7f-fb3a5838cd27'
        APP_NAME = 'learnjeankinsapp'
        AWS_DEFAULT_REGION = "eu-north-1"
        AWS_ECS_CLUSTER = 'LeanJenkins-prod'
        AWS_ECS_SERVICES = 'LearnJenkins-Service-prod'
        AWS_ECS_TD = 'LeanrJenkins-task-def-prod'
        AWS_DOCKER_REGISTRY = '182399711075.dkr.ecr.eu-north-1.amazonaws.com'
    }
    stages {
        stage('Build') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                  echo 'test trigger'
                  ls -la
                  node --version
                  npm --version
                  npm ci
                  npm run build
                  '''
            }
        }
        stage ('Build Docker Image') {
            agent {
                docker {
                    image 'my-aws-cli'
                    reuseNode true
                    args "-u root -v /var/run/docker.sock:/var/run/docker.sock --entrypoint=''"
                }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'my-aws', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) 
                {
                sh '''
                docker build -t $AWS_DOCKER_REGISTRY/$APP_NAME:$REACT_APP_VERSION .
                aws ecr get-login-password | docker login --username AWS --password-stdin $AWS_DOCKER_REGISTRY
                docker push $AWS_DOCKER_REGISTRY/$APP_NAME:$REACT_APP_VERSION
                '''
            }
        }
        }
        stage ('deploy AWS') {
            agent {
                docker {
                    image 'my-aws-cli'
                    reuseNode true
                    args "-u root --entrypoint=''"
                }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'my-aws', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                sh '''
                    aws --version
                    yum install jq -y
                    LATEST_TD_REVISION=$(aws ecs register-task-definition --cli-input-json file://aws/task-definition-prod.json | jq '.taskDefinition.revision')
                    aws ecs update-service --cluster $AWS_ECS_CLUSTER --service $AWS_ECS_SERVICES --task-definition $AWS_ECS_TD:$LATEST_TD_REVISION
                    #aws ecs wait services-stable --cluster $AWS_ECS_CLUSTER --services $AWS_ECS_SERVICES
                '''
                }
            }
        }
        stage ('Stage Tests') {
            parallel {
                stage('Unit Test') {
                    agent {
                        docker {
                            image 'node:18-alpine'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                        test -f build/index.html
                        npm test
                        '''
                    }
                        post {
                            always {
                                junit 'jest-results/junit.xml'
                            }
                        }
                }
                stage('E2E') {
                    agent {
                        docker {
                            image 'my-playwright'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            serve -s build &
                            sleep 15
                            npx playwright test --reporter=html
                        '''
                    }
                        post {
                            always {
                                publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playright Local Report', reportTitles: '', useWrapperFileDirectly: true])
                             }
                        }
                }
            }
        }

        stage('Deploy staging') {
            agent {
                docker {
                    image 'my-playwright'
                    reuseNode true
                        }
                    }
            environment {
                CI_ENVIRONMENT_URL = 'STAGING_URL'
            }
            steps {
                sh '''
                  netlify --version
                  echo "Deploying to production site ID:$NETLIFY_SITE_ID"
                  netlify status
                  netlify deploy --dir=build --json > deploy-output.json
                  CI_ENVIRONMENT_URL=$(jq -r '.deploy_url' deploy-output.json)
                  npx playwright test --reporter=html
                    '''
                    }
            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Staging E2E', reportTitles: '', useWrapperFileDirectly: true])
                        }
                        }
                }

        stage('Deploy Prod') {
            agent {
                docker {
                    image 'my-playwright'
                    reuseNode true
                        }
                    }
            environment {
                CI_ENVIRONMENT_URL = 'https://bucolic-khapse-10c2e9.netlify.app'

                }
            steps {
                sh '''
                  netlify --version
                  echo "Deploying to production site ID:$NETLIFY_SITE_ID"
                  netlify status
                  netlify deploy --dir=build --prod
                  npx playwright test --reporter=html
                    '''
                    }
            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Prod E2E', reportTitles: '', useWrapperFileDirectly: true])
                        }
                        }
                }
        }
}

