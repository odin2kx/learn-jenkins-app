pipeline {
    agent any
    environment {
        NETLIFY_SITE_ID = '6f30e67b-90a6-40ee-bc7f-fb3a5838cd27'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
        REACT_APP_VERSION = "1.0.$BUILD_ID"
    }
    stages {
        stage ('Docker') {
            steps {
                sh 'docker build -t my-playwright .'
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
                            image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            npm install serve
                            node_modules/.bin/serve -s build &
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
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                        }
                    }
            environment {
                CI_ENVIRONMENT_URL = 'STAGING_URL'
            }
            steps {
                sh '''
                  npm install netlify-cli node-jq
                  node_modules/.bin/netlify --version
                  echo "Deploying to production site ID:$NETLIFY_SITE_ID"
                  node_modules/.bin/netlify status
                  node_modules/.bin/netlify deploy --dir=build --json > deploy-output.json
                  CI_ENVIRONMENT_URL=$(node_modules/.bin/node-jq -r '.deploy_url' deploy-output.json)
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
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                        }
                    }
            environment {
                CI_ENVIRONMENT_URL = 'https://bucolic-khapse-10c2e9.netlify.app'

                }
            steps {
                sh '''
                  npm install netlify-cli
                  node_modules/.bin/netlify --version
                  echo "Deploying to production site ID:$NETLIFY_SITE_ID"
                  node_modules/.bin/netlify status
                  node_modules/.bin/netlify deploy --dir=build --prod
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

