pipeline {
    agent any
    tools {
        maven 'maven'
    }
    environment{
        IMAGE_NAME ='project-1'
        IMAGE_TAG = 'latest'
        TENET_ID = 'f4220ae5-c90d-4d8f-8ee4-a76c325d6b28'
        ACR_NAME = 'springbootproject1'
        ACR_LOGIN_SERVER = 'springbootproject1.azurecr.io'
        FULL_IMAGE_NAME = "${ACR_LOGIN_SERVER}/${IMAGE_NAME}:${IMAGE_TAG}"
        RG = 'harit-rg'
        AKS_NAME = 'demo-aks' 
    }
    stages {
        stage('Checkout from Git') {
            steps {
                git branch: 'prod', url: 'https://github.com/harit2882/enahanced-petclinc-springboot.git'
            }
        }
        stage('Validate with Maven') {
            steps {
                sh 'mvn clean validate'
                echo 'Validation Successful'
            }
        }
        stage('Compile with Maven') {
            steps {
                sh 'mvn clean compile'
                echo 'Compilation Successful'
            }
        }
        stage('Sonar Analysis') {
            environment {
                SCANNER_HOME = tool 'SonarScanner'
            }
            steps {
                withSonarQubeEnv('sonarserver') {
                    sh "${SCANNER_HOME}/bin/sonar-scanner \
                     -Dsonar.projectKey=harit2882_project-1 \
                     -Dsonar.organization=harit2882 \
                     -Dsonar.projectName=project-1 \
                     -Dsonar.java.binaries=."
                }
            }
        }
        stage('Maven Package') {
            steps {
                sh 'mvn clean package'
                echo 'Packaging Successful'
            }
        }
        stage('Sonar Quality Gate') {
            steps {
                timeout(time: 1, unit: 'MINUTES') {
                   waitForQualityGate abortPipeline: true, credentialsId: 'sonar'
                }
            }
        }
        stage('Docker Build') {
            steps {
                script {
                    echo 'Building Docker Image.........'
                    docker.build ("${IMAGE_NAME}:${IMAGE_TAG}")
                }
            }
        }
        stage('Azure Login TO ACR') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'azure-jenkins', passwordVariable: 'AZURE_PASSWORD', usernameVariable: 'AZURE_USERNAME')]) {
                    script {   
                        echo 'Azure Login Started........'
                        sh '''
                        az login --service-principal -u $AZURE_USERNAME -p $AZURE_PASSWORD --tenant $TENET_ID
                        az acr login --name $ACR_NAME
                    '''
                    }
                }
            }
        }
        stage('Docker Push to ACR') {
            steps {
                script {
                    echo 'Pushing Docker Image to ACR........'
                    sh '''
                        docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${FULL_IMAGE_NAME}
                        docker push ${FULL_IMAGE_NAME}
                    '''
                }
            }
        }
        stage('Azure Login To AKS') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'azure-jenkins', passwordVariable: 'AZURE_PASSWORD', usernameVariable: 'AZURE_USERNAME')]) {
                    script {   
                        echo 'Azure Login Started to AKS'
                        sh '''
                        az login --service-principal -u $AZURE_USERNAME -p $AZURE_PASSWORD --tenant $TENET_ID
                        az aks get-credentials --resource-group $RG --name $AKS_NAME --overwrite-existing
                    '''
                    }
                }
            }
        }
        stage('Deploy to AKS') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'azure-jenkins', passwordVariable: 'AZURE_PASSWORD', usernameVariable: 'AZURE_USERNAME')]) {
                    script {
                        echo 'Deploying Application to AKS........'
                        sh '''
                            az login --service-principal -u $AZURE_USERNAME -p $AZURE_PASSWORD --tenant $TENET_ID
                            kubectl apply -f k8s/sprinboot-deployment.yaml
                        '''
                    }
                }
            }
        }
    }
}