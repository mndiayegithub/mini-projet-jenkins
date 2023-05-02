/* import shared library */
@Library('jenkins-shared-library') _

pipeline {
    environment {
        IMAGE_NAME = "static-website"
        IMAGE_TAG = "latest"
        IMAGE_PORT = 8080
        STAGING = "environment-staging"
        PRODUCTION = "environment-production"
        DOCKERHUB_USERNAME = "mndiayepro97"
        DOCKERHUB_CREDENTIALS = credentials('dockerhub_credentials')  
    }
    agent none 
    stages {
        stage ('Build image') {
            agent any
            steps {
                script {
                    sh '''
                        echo "Build image"
                        docker build -t $DOCKERHUB_USERNAME/$IMAGE_NAME:$IMAGE_TAG .
                        docker run --name $IMAGE_NAME -d -p 80:$IMAGE_PORT -e PORT=$IMAGE_PORT --network jenkins_default $DOCKERHUB_USERNAME/$IMAGE_NAME:$IMAGE_TAG
                        sleep 5s
                        '''
                }
            }
        }
        
        stage('Scan Image with  SNYK') {
            agent any
            environment{
                SNYK_TOKEN = credentials('snyk_token')
            }
            steps {
                script{
                    sh '''
                    echo "Starting Image scan ${DOCKERHUB_ID}/$IMAGE_NAME:$IMAGE_TAG ..." 
                    echo There is Scan result : 
                    SCAN_RESULT=$(docker run --rm -e SNYK_TOKEN=$SNYK_TOKEN -v /var/run/docker.sock:/var/run/docker.sock -v $(pwd):/app snyk/snyk:docker snyk test --docker $DOCKERHUB_ID/$IMAGE_NAME:$IMAGE_TAG --json ||  if [[ $? -gt "1" ]];then echo -e "Warning, you must see scan result \n" ;  false; elif [[ $? -eq "0" ]]; then   echo "PASS : Nothing to Do"; elif [[ $? -eq "1" ]]; then   echo "Warning, passing with something to do";  else false; fi)
                    echo "Scan ended"
                    '''
                }
            }
        }

        stage ('Test image') {
            agent any
            steps {
                script {
                    sh '''
                        curl -X GET http://$IMAGE_NAME:$IMAGE_PORT | grep -q "Welcome"
                    '''
                }
            }
        }

        stage ('Clean container') {
            agent any
            steps {
                script {
                    sh '''
                        docker stop $IMAGE_NAME
                        docker rm $IMAGE_NAME
                    '''
                }
            }
        }

        stage ('Login to Docker Hub') {
            agent any
            steps {
                script {
                    sh '''
                        echo $DOCKERHUB_CREDENTIALS_PSW | sudo docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin
                        echo 'Login Completed' 
                    '''
                }
            }
        }
        
        stage ('Push image to Docker Hub') {
            agent any
            steps {
                script {
                    sh '''
                        echo "Pushing image to Docker hub"
                        docker push $DOCKERHUB_USERNAME/$IMAGE_NAME:$IMAGE_TAG
                        echo 'Push image completed' 
                    '''
                }
            }
        }

  
        stage ('Push image in staging and deploy it') {
            when {
                expression { GIT_BRANCH == 'origin/master' }
            }
                agent any
                environment {
                    HEROKU_API_KEY = credentials('api_key_heroku')
                }
                steps {
                    script {
                        sh '''
                            echo "Pushing image into staging"
                            heroku container:login
                            heroku create $STAGING || echo "project already exists"
                            heroku container:push -a $STAGING web
                            heroku container:release -a $STAGING web
                        '''
                    }
                }
        }

        stage ('Push image in production and deploy it') {
            when {
                expression { GIT_BRANCH == 'origin/master' }
            }
                agent any
                environment {
                    HEROKU_API_KEY = credentials('api_key_heroku')
                }
                steps {
                    script {
                        sh '''
                            echo "Pushing image into production"
                            heroku container:login
                            heroku create $PRODUCTION || echo "project already exists"
                            heroku container:push -a $PRODUCTION web
                            heroku container:release -a $PRODUCTION web
                        '''
                    }
                }
        }
    }

    post {
        always {
            script {
                slackNotifier currentBuild.result
            }
        }
    }
}
