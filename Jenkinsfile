pipeline {
    agent any

    environment {
        REGISTRY = 'user12.azurecr.io'
        IMAGE_NAME = 'delivery'
        AKS_CLUSTER = 'user12-aks'
        RESOURCE_GROUP = 'user12-rsrcgrp'
        AKS_NAMESPACE = 'default'
        AZURE_CREDENTIALS_ID = 'Azure-Cred'
        GIT_CREDENTIALS_ID = 'Git-Cred'
        TENANT_ID = 'f46af6a3-e73f-4ab2-a1f7-f33919eda5ac' // Service Principal 등록 후 생성된 ID
    }
 
    stages {
        stage('Clone Repository') {
            steps {
                checkout scm
            }
        }

        stage('Check Changes') {
            steps {
                script {
                    def changes = sh(script: "git diff --name-only HEAD~1", returnStdout: true).trim()
                    def ignoredFiles = ['./kubernetes/deploy.yaml', './kubernetes/service.yaml']
                    
                    def shouldRun = true
                    changes.split('\n').each { file ->
                        if (ignoredFiles.any { file.matches(it) }) {
                            shouldRun = false
                        }
                    }
                    
                    if (!shouldRun) {
                        echo "k8s yaml files were changed. Skipping pipeline."
                        currentBuild.result = 'SUCCESS'
                        return
                    }
                }
            }
        }

        stage('Maven Build') {
            steps {
                withMaven(maven: 'Maven') {
                    sh 'mvn package -DskipTests'
                }
            }
        }
        
        stage('Docker Build') {
            steps {
                script {
                    image = docker.build("${REGISTRY}/${IMAGE_NAME}:v${env.BUILD_NUMBER}")
                }
            }
        }
        
        stage('Azure Login') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: env.AZURE_CREDENTIALS_ID, usernameVariable: 'AZURE_CLIENT_ID', passwordVariable: 'AZURE_CLIENT_SECRET')]) {
                        sh 'az login --service-principal -u $AZURE_CLIENT_ID -p $AZURE_CLIENT_SECRET --tenant ${TENANT_ID}'
                    }
                }
            }
        }
        
        stage('Push to ACR') {
            steps {
                script {
                    sh "az acr login --name ${REGISTRY.split('\\.')[0]}"
                    sh "docker push ${REGISTRY}/${IMAGE_NAME}:v${env.BUILD_NUMBER}"
                }
            }
        }
        
        stage('CleanUp Images') {
            steps {
                sh """
                docker rmi ${REGISTRY}/${IMAGE_NAME}:v$BUILD_NUMBER
                """
            }
        }
        
        stage('Update deploy.yaml / service.yaml') {
            steps {
                script {
                    sh "az aks get-credentials --resource-group ${RESOURCE_GROUP} --name ${AKS_CLUSTER}"
                    sh """
                    sed 's/IMAGE_TAG/v${env.BUILD_ID}/g' template/deploy.yaml > kubernetes/deploy.yaml
                    cat template/service.yaml > kubernetes/service.yaml
                    """
                }
            }
        }

        stage('Push .yaml to Repo') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: env.GIT_CREDENTIALS_ID, usernameVariable: 'GIT_USER', passwordVariable: 'GIT_SECRET')]) {
                        sh '''
                        git config --global user.email ''
                        git config --global user.name 'Jenkins'
                        git add kubernetes/deploy.yaml kubernetes/service.yaml
                        git commit -m "Update deploy.yaml / service.yaml"
                        git push https://${GIT_SECRET}@github.com/tekkenlog/reqres_delivery.git master
                        '''
                    }
                }
            }
        }
    }
}
