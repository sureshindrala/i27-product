pipeline {
    agent {
        label "k8s-slave"
    }
    parameters {
        choice(name: 'buildOnly',
        choices: 'no\nyes',
        description: 'This will only build the application'
        )
        choice (name: 'scanOnly',
        choices: 'no\nyes',
        description: 'This will only build the application'
        )
        choice(name: 'dockerPush',
        choices: 'no\nyes',
        description: 'This will only push the docker image'
        )
        choice(name:'deployToDev',
        choices: 'no\nyes',
        description: 'This will deploy dev environment'
        )
        choice (name: 'deployToTest',
        choices: 'no\nyes',
        description: 'This will deploy Test environment'
        )
        choice(name: 'deployToStage',
        choices: 'no\nyes',
        description: 'This will deploy Stage environment'
        )
        choice(name: 'deployToProd',
        choices: 'no\nyes',
        description: 'This will deploy prod environment'
        )
    }
    environment {
        APPLICATION_NAME = "product"
        POM_VERSION = readMavenPom().getVersion()
        POM_PACKAGING = readMavenPom().getPackaging()
        DOCKER_HUB = "docker.io/sureshindrala"
        DOCKER_CREDS = credentials("dockerhub_creds")
        SONAR_URL = "http://34.66.190.70:9000/"
        SONAR_TOKEN = credentials('sonar_creds')
    }
    tools {
        maven 'Maven-3.8.8'
        jdk 'Jdk-17'
    }
    stages {
        stage('Build') {
            when {
                anyOf {
                    expression {
                        params.buildOnly == 'yes'
                    }
                }
            }
            steps {
                script {
                   buildApp.call()
                }
                
               // echo "Building ${env.APPLICATION_NAME} application"
               // sh 'mvn clean package -DskipTests=true'
            }
        }
        stage('Unit Test') {
            when {
                anyOf {
                    expression {
                        params.buildOnly == 'yes'
                        params.dockerPush == 'yes'
                    }
                }
            }
            steps {
                echo "Running unit tests for ${env.APPLICATION_NAME} application"
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }
        stage('sonar') {
            when {
                anyOf {
                    expression {
                        params.scanOnly == 'yes'
                    }
                }
            }
            steps {
                echo "******starting with sonarqube********"
                withSonarQubeEnv('SonarQube') {
                    sh '''
                    echo "Starting SonarQube analysis"
                    mvn clean verify sonar:sonar \
                        -Dsonar.projectKey=i27-product \
                        -Dsonar.host.url=${SONAR_URL} \
                        -Dsonar.login=${SONAR_TOKEN}
                    '''
                }
                timeout(time: 2, unit: 'MINUTES') {
                    script {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
        }
        stage('Docker Build') {
            when {
                anyOf {
                    expression {
                        params.dockerPush == 'yes'
                    }
                }
            }
            steps {
                script {
                  dockerBuildandpush().call()
                }
               
            }
        }
        stage('Deploy to Dev') {
            when {
                expression {
                    params.deployToDev == 'yes'
                }
            }
            steps {
                script {
                    imageValidation().call()
                    echo "***** Entering Dev Environment *****"
                    dockerDeploy('dev', '5132', '8132').call()
                
                }
            }
        }
        stage('Deploy to Test') {
            when {
                expression {
                    params.deployToTest == 'yes'
                }
            }
            steps {
                script {
                    imageValidation().call()
                    echo "***** Entering Test Environment *****"
                    dockerDeploy('tst', '6132', '8132').call()
                    echo "*********** Test Environment successfully *******"
                }
            }
        }
        stage('Deploy to Stage') {
            when{
                expression {
                    params.deployToStage == 'yes'
                }
            }
            steps {
                script {
                    imageValidation().call()
                    echo "***** Entering Stage Environment *****"
                    dockerDeploy('stage', '7132', '8132').call()
                    echo " stage environment completed suceesfully"
                }
            }
        }
        stage('Deploy to Prod') {
            when {
                allOf {
                    anyOf {
                    expression {
                        params.deployToProd =='yes'
                    }              
                }
                anyOf {
                    branch 'release/*'
                }
                
                }
            }
            steps {
                timeout(time:300, unit: 'SECONDS') {
                    input message: "Deploying ${env.APPLICATION_NAME} to prod ????", ok: 'yes' , submitter: 'greeshma'
                }
                script {
                    imageValidation().call()
                    dockerDeploy('prod', '8132', '8132').call()
                 
                }
            }
        }
        stage('clean') {
            steps { 
                cleanWs()
            }
        }
    }
  }


def dockerBuildandpush(){
    return {

            sh """
                cp ${WORKSPACE}/target/i27-${APPLICATION_NAME}-${POM_VERSION}.${POM_PACKAGING} ./.cicd
                ls -la ./.cicd
                echo "***********Building Docker Image*******************"
                docker build --force-rm --no-cache --pull --rm=true --build-arg JAR_SOURCE=i27-${APPLICATION_NAME}-${POM_VERSION}.${POM_PACKAGING} -t ${DOCKER_HUB}/${APPLICATION_NAME}:${GIT_COMMIT} ./.cicd
                docker images
                echo "************Docker login*******************"
                docker login -u ${DOCKER_CREDS_USR} -p ${DOCKER_CREDS_PSW}
                echo "**************Docker Push******************"
                docker push ${DOCKER_HUB}/${APPLICATION_NAME}:${GIT_COMMIT}
                echo "********** pushed image successfully !!!! **********"
             """   
            }
}


def dockerDeploy(envDeploy, hostPort, contPort) {
    return {
    echo "******************************** Deploying to $envDeploy Environment ********************************"
    withCredentials([usernamePassword(credentialsId: 'docker_env_creds', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) {
        // some block
        // With the help of this block, ,the slave will be connecting to docker-vm and execute the commands to create the containers.
        //sshpass -p ssh -o StrictHostKeyChecking=no user@host command_to_run
        //sh "sshpass -p ${PASSWORD} -v ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} hostname -i" 
        
    script {
        // Pull the image on the Docker Server
        sh "sshpass -p ${PASSWORD} -v ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} docker pull ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}"
        
        try {
            // Stop the Container
            echo "Stoping the Container"
            sh "sshpass -p ${PASSWORD} -v ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} docker stop ${env.APPLICATION_NAME}-$envDeploy"

            // Remove the Container 
            echo "Removing the Container"
            sh "sshpass -p ${PASSWORD} -v ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} docker rm ${env.APPLICATION_NAME}-$envDeploy"
             } catch(err) {
            echo "Caught the Error: $err"
        }

        // Create a Container 
        echo "Creating the Container"
        sh "sshpass -p ${PASSWORD} -v ssh -o StrictHostKeyChecking=no ${USERNAME}@${docker_server_ip} docker run -d -p $hostPort:$contPort --name ${env.APPLICATION_NAME}-$envDeploy ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}"
        }
    }
    }
    
}
def imageValidation() {
    return {
        println ("Pulling the docker image")
        try {
        sh "docker pull ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}"
        }
        catch (Exception e) {
            println ("OOPS..!, Docker images with this tag is not availiable ")
            buildApp().call()
            dockerBuildandpush().call()
            }
        }
    }

def buildApp() {
        return {
            echo "Building ${env.APPLICATION_NAME} application "
            sh "mvn clean package -DskipTests=true"
        }
    }



// User, container port is 8232
// dev 5232
// test 6232
//stage 7232
//prod 8232



