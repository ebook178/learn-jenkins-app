pipeline {
    agent any
    
    environment{
        REACT_APP_VERSION = "1.0.$BUILD_ID"
        APP_NAME = 'learnjenkinsapp'
        AWS_DEFAULT_REGION = 'ap-southeast-2'
        AWS_DOCKER_REGISTRY = '739275451107.dkr.ecr.ap-southeast-2.amazonaws.com'
        AWS_ECS_CLUSTER = 'LearnJenkinsApp-Cluster-Prod'
        AWS_ECS_SERVICE_PROD = 'LearnJenkinsApp-Service-Prod'
        AWS_ECS_TD_PROD = 'LearnJenkinsApp-TaskDefinition-Prod'
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

       stage('Build Docker Image') {
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
                        docker build -f Dockerfile -t $AWS_DOCKER_REGISTRY/$APP_NAME:$REACT_APP_VERSION .
                        aws ecr get-login-password | docker login --username AWS --password-stdin $AWS_DOCKER_REGISTRY
                        docker push $AWS_DOCKER_REGISTRY/$APP_NAME:$REACT_APP_VERSION
                    '''
                  }

            }
       }

       stage('Deploy to AWS') {
           agent {
                docker{
                    image 'my-aws-cli'
                    reuseNode true
                    args "--entrypoint=''"
                }
           }

           steps {
                withCredentials([usernamePassword(credentialsId: 'my-aws', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                    sh '''
                        aws --version
                        LATEST_TD_REVISION=$(aws ecs register-task-definition --cli-input-json file://aws/task-definiation_prod.json | jq '.taskDefinition.revision')
                        echo $LATEST_TD_REVISION
                        aws ecs update-service --cluster $AWS_ECS_CLUSTER --service $AWS_ECS_SERVICE_PROD --task-definition $AWS_ECS_TD_PROD:$LATEST_TD_REVISION
                        aws ecs wait services-stable --cluster $AWS_ECS_CLUSTER --services $AWS_ECS_SERVICE_PROD
                    '''
                }
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
