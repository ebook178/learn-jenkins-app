pipeline {
    agent any
    
    environment{
        REACT_APP_VERSION = "1.0.$BUILD_ID"
    }

    stages {

       stage('Deploy to AWS') {
           agent {
                docker{
                    image 'amazon/aws-cli'
                    reuseNode true
                    args "--entrypoint=''"
                }
           }
           environment {

           }
           steps {
                withCredentials([usernamePassword(credentialsId: 'my-aws', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                    sh '''
                        aws --version
                        aws ecs register-task-definition --cli-input-json file://aws/task-definiation_prod.json
                    '''
                }
           }
       }   

       stage('Docker') {
         steps {
            sh 'docker build -t my-playwright -f Dockerfile .'
         }
       }

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

        stage('Deploy Stage') {
            agent{
                docker{
                    image 'my-playwright'
                    reuseNode true
                }
            }
        
            environment {
                CI_ENVIRONMENT_URL = 'STAGING_URL_TO_BE_SET'
            }

            steps {
                sh '''
                    node --version
                    netlify --version
                    echo "Deploying to prodution. Site ID:$NETLIFY_SITE_ID"
                    netlify status
                    netlify deploy --dir=build --json > deploy-output.json
                    CI_ENVIRONMENT_URL=$(jq -r '.deploy_url' deploy-output.json)
                    npx playwright test --reporter=html
                '''
            }
            
            post{
                always{
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Deploy Stage', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }


        stage('Deploy prod') {
            agent{
                docker{
                    image 'my-playwright'
                    reuseNode true
                }
            }
            environment{
                CI_ENVIRONMENT_URL = "https://leafy-cupcake-8682ce.netlify.app"
            }
            steps {
                sh '''
                    node --version
                    netlify --version
                    echo "Deploying to prodution. Site ID:$NETLIFY_SITE_ID"
                    netlify status
                    netlify deploy --dir=build --prod                   
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
