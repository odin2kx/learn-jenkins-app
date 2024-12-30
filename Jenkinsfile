pipeline {
    agent any
    environment {

        REACT_APP_VERSION = "1.0.$BUILD_ID"
        AWS_DEFAULT_REGION = "eu-north-1"
    }
    stages {
        stage ('deploy AWS') {
            agent {
                docker {
                    image 'amazon/aws-cli'
                    reuseNode true
                    args "--entrypoint=''"
                }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'my-aws', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                sh '''
                    aws --version
                    aws ecs register-task-definition --cli-input-json file://aws/task-definition-prod.json
                '''
                }
            }
        }
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

