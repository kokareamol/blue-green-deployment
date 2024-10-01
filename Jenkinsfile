pipeline {
    agent any
    
    parameters {
        choice(name: 'DEPLOY_ENV', choices: ['blue', 'green'], description: 'Choose which environment to deploy: Blue or Green')
        choice(name: 'DOCKER_TAG', choices: ['blue', 'green'], description: 'Choose the Docker image tag for the deployment')
        booleanParam(name: 'SWITCH_TRAFFIC', defaultValue: false, description: 'Switch traffic between Blue and Green')
    }
    
    tools{
        jdk 'jdk17'
        maven 'maven3'
    }
    
    environment{
        SCANNER_HOME= tool 'sonar-scanner'
        IMAGE_NAME = "kokareamol/bankapp"
        TAG = "${params.DOCKER_TAG}"
        KUBE_NAMESPACE = 'webapps'
    }

    stages {
        stage('git checkout') {
            steps {
                checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[credentialsId: 'github-token', url: 'https://github.com/kokareamol/blue-green-deployment.git']])
            }
        }
        stage('mvn compile') {
            steps {
                sh "mvn compile "
            }
        }
        
        // stage('trivy fs scan') {
        //     steps {
        //         sh "trivy fs --format table -o fs.txt ."
        //     }
        // }
        
        stage('sonar scanning') {
            steps {
                withSonarQubeEnv('sonar-server') {
                   sh "$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectKey=multitier -Dsonar.projectName=multitier -Dsonar.java.binaries=target"
               }
            }
        }
        
        stage('quality gates') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: false
               }
            }
        }
        
        stage('mvn package/build') {
            steps {
                sh "mvn package -DskipTests=true "
            }
        }
        
        stage('mvn deploy/publish nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'maven-settings', jdk: '', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                   sh "mvn deploy -DskipTests=true "
                }
            }
        }
        
        stage('docker build and tag') {
            steps {
                script{
                    withDockerRegistry(credentialsId: 'docker-token') {
                        sh "docker build -t ${IMAGE_NAME}:${TAG} ."
                    }
                }
            }
        }
        
        // stage('trivy image scan') {
        //     steps {
        //         sh "trivy image --format table -o dm.html ${IMAGE_NAME}:${TAG}"
        //     }
        // }
        
        stage('docker img push') {
            steps {
                script{
                    withDockerRegistry(credentialsId: 'docker-token') {
                        sh "docker push ${IMAGE_NAME}:${TAG}"
                    }
                }
            }
        }
        
        stage('Deploy MySQL Deployment and Service') {
            steps {
                script {
                    withKubeConfig(caCertificate: '', clusterName: 'EKS_CLOUD', contextName: '', credentialsId: 'k8-secret', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://91DABD3A759328CBA176DEE2E314B78A.gr7.ap-south-1.eks.amazonaws.com') {
                        sh "kubectl apply -f mysql-ds.yml -n ${KUBE_NAMESPACE}"  // Ensure you have the MySQL deployment YAML ready
                    }
                }
            }
        }
        
        stage('Deploy SVC-APP') {
            steps {
                script {
                    withKubeConfig(caCertificate: '', clusterName: 'EKS_CLOUD', contextName: '', credentialsId: 'k8-secret', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://91DABD3A759328CBA176DEE2E314B78A.gr7.ap-south-1.eks.amazonaws.com') {
                        sh """ if ! kubectl get svc bankapp-service -n ${KUBE_NAMESPACE}; then
                                kubectl apply -f bankapp-service.yml -n ${KUBE_NAMESPACE}
                              fi
                        """
                   }
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    def deploymentFile = ""
                    if (params.DEPLOY_ENV == 'blue') {
                        deploymentFile = 'app-deployment-blue.yml'
                    } else {
                        deploymentFile = 'app-deployment-green.yml'
                    }

                    withKubeConfig(caCertificate: '', clusterName: 'EKS_CLOUD', contextName: '', credentialsId: 'k8-secret', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://91DABD3A759328CBA176DEE2E314B78A.gr7.ap-south-1.eks.amazonaws.com') {
                        sh "kubectl apply -f ${deploymentFile} -n ${KUBE_NAMESPACE}"
                    }
                }
            }
        }
        
        stage('Switch Traffic Between Blue & Green Environment') {
            when {
                expression { return params.SWITCH_TRAFFIC }
            }
            steps {
                script {
                    def newEnv = params.DEPLOY_ENV

                    // Always switch traffic based on DEPLOY_ENV
                    withKubeConfig(caCertificate: '', clusterName: 'EKS_CLOUD', contextName: '', credentialsId: 'k8-secret', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://46743932FDE6B34C74566F392E30CABA.gr7.ap-south-1.eks.amazonaws.com') {
                        sh '''
                            kubectl patch service bankapp-service -p "{\\"spec\\": {\\"selector\\": {\\"app\\": \\"bankapp\\", \\"version\\": \\"''' + newEnv + '''\\"}}}" -n ${KUBE_NAMESPACE}
                        '''
                    }
                    echo "Traffic has been switched to the ${newEnv} environment."
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    def verifyEnv = params.DEPLOY_ENV
                    withKubeConfig(caCertificate: '', clusterName: 'EKS_CLOUD', contextName: '', credentialsId: 'k8-secret', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://91DABD3A759328CBA176DEE2E314B78A.gr7.ap-south-1.eks.amazonaws.com') {
                        sh """
                        kubectl get pods -l version=${verifyEnv} -n ${KUBE_NAMESPACE}
                        kubectl get svc bankapp-service -n ${KUBE_NAMESPACE}
                        """
                    }
                }
            }
        }
        
    }
}
