pipeline {
    agent any
    
    environment{
        NETLIFY_SITE_ID = "b1d462e5-38e4-45c0-ade7-5c49ebd133cd"
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
    }

    stages {
        stage('Build') {
            agent{
                docker{
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    echo "Small Change"
                    ls -la
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    ls -la
               '''
            }
        }

        stage('Test') {
            parallel {
                stage('Unit test') {
                    agent{
                        docker{
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
                    post{
                        always{
                            junit 'jest-results/junit.xml'
                        }
                    }                    
                }

                stage('E2E') {
                    agent{
                        docker{
                            image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            npm install serve
                            node_modules/.bin/serve -s build &
                            sleep 10
                            npx playwright test --reporter=html
                        '''
                    }
                    post{
                        always{
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright Local', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }
            }
        }


        stage('Deploy Stage') {
            agent{
                docker{
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }
        
            environment {
                CI_ENVIRONMENT_URL = 'STAGING_URL_TO_BE_SET'
            }

            steps {
                sh '''
                    node --version
                    npm install netlify-cli node-jq
                    node_modules/.bin/netlify --version
                    echo "Deploying to prodution. Site ID:$NETLIFY_SITE_ID"
                    node_modules/.bin/netlify status
                    node_modules/.bin/netlify deploy --dir=build --json > deploy-output.json
                    CI_ENVIRONMENT_URL=$(node_modules/.bin/node-jq -r '.deploy_url' deploy-output.json)
                    npx playwright test --reporter=html
                '''
            }
            
            post{
                always{
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Deploy Stage', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }

        stage('Approval') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    input message: 'Do you wish to deploy to production?', ok: 'Yes, I am sure!'
                }
            }
        }

        stage('Deploy prod') {
            agent{
                docker{
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }
            environment{
                CI_ENVIRONMENT_URL = "https://leafy-cupcake-8682ce.netlify.app"
            }
            steps {
                sh '''
                    node --version
                    npm install netlify-cli
                    node_modules/.bin/netlify --version
                    echo "Deploying to prodution. Site ID:$NETLIFY_SITE_ID"
                    node_modules/.bin/netlify status
                    node_modules/.bin/netlify deploy --dir=build --prod                   
                    npx playwright test --reporter=html
                '''
            }
            post{
                always{
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Pro E2E', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }

    }
}
