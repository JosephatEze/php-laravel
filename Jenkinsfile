pipeline {

       // gent {
           //docker{ 
            //   image 'php:7.4-cli'
          // }
        //}
        agent any
        options {
                //skipStagesAfterUnstable()
                disableConcurrentBuilds()
                parallelsAlwaysFailFast()
        }
        environment {
            AWS_ACCESS_KEY_ID     = credentials('jenkins-aws-secret-key-id')
            AWS_SECRET_ACCESS_KEY = credentials('jenkins-aws-secret-access-key')
            DOCKER_PASSWORD =  credentials('docker-hub-password')
            DOCKER_registry= 'josephat'

            ecrRepository = "004510841606.dkr.ecr.us-east-1.amazonaws.com/php-app"
            NAME = "php"
        }
        
        stages{
        stage('checkout SCM') {
            steps {
                  //checkout([$class: 'GitSCM', branches: [[name: '*/Dev']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[url: 'https://github.com/reliancehealthinc/jenkins-templates.git']]])
                   checkout([$class: 'GitSCM', branches: [[name: 'stable-2.x']], userRemoteConfigs: [[credentialsId:  'my-ssh-private-key-id', url: 'ssh://github.com/jenkinsci/git-plugin.git']]])   
              }
        }
        stage('Install Composer') {
            steps {
                sh 'composer install'
                //sh 'sudo apt-get install php-cli'
                //sh 'sudo apt-get install phpunit'
            }
        }
        stage("PHPLint") {
            steps {
            sh 'find app -name "*.php" -print0 | xargs -0 -n1 php -l'
            }
        }
        stage("PHPUnit") {
            steps {
            sh 'vendor/phpunit/phpunit/phpunit --bootstrap build/bootstrap.php --configuration phpunit-coverage.xml'
            //sh './vendor/bin/phpunit --bootstrap ./vendor/autoload.php ./tests/quotetest.php'
            }
        }
        stage('Build docker image'){
            steps{
                    sh "docker build -t ${NAME} -f ./Dockerfile ."
                
            }
        }
        //FOR AWS ECR 
        stage('push image to ECR'){
            steps{
                withAWS(region: 'eu-west-1', role: 'jenkins') {
                    
                        sh "aws ecr describe-repositories --repository-names ${NAME} || aws ecr create-repository --repository-name ${NAME} --image-scanning-configuration scanOnPush=true"
                        sh "aws ecr get-login-password | docker login --password-stdin --username AWS '${ecrRepository}/${NAME}'"
                        //sh "docker build -t ${NAME} -f ./Dockerfile ."
                        sh "docker tag ${NAME} ${ecrRepository}/${NAME}:${BUILD_NUMBER}"
                        sh "docker push ${ecrRepository}/${NAME}:${BUILD_NUMBER}"
                  }
            }
        }
        //FOR DOCKER HUB
        stage('Push image to docker Hub'){
            steps{
                script{
                   withCredentials([string(credentialsId: 'dockerhub-pwd', variable: 'dockerhubpwd')]) {
                   sh "docker login -u ${docker-hub-username} -p ${dockerhubpwd}"
                   sh "docker push ${DOCKER_registry}/${NAME}:${BUILD_NUMBER}"
                    }
                   
                }
            }
        }
        //stage('containerize app') {
          //  steps {
            //    sh 'echo "${DOCKER_PASSWORD}" | docker login -u "${DOCKER_USERNAME}" --password-stdin' 
                // Build and tag for internal registry
              //  sh "docker build -t ${INTERNAL_IMAGE_NAME} ."
                //sh 'docker tag ${INTERNAL_IMAGE_NAME} ${PUBLIC_IMAGE_NAME}:${BUILD_NUMBER}'
            //}
        //}
        stage('deploy app') {
            when {
              expression {
                currentBuild.result == null || currentBuild.result == 'SUCCESS' 
              }
            }
            steps {
                sh 'aws eks --region us-east-1 update-kubeconfig --name eks-cluster-name'
                sh ' kubectl apply -f ~/k8s/templates/customer-service-deploy.yml'
                //sh 'helm install [app-name] [chart]'
            }
        }

    }
}
