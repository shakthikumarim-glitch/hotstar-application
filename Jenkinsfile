pipeline {
    agent any

    tools {
        jdk 'jdk'
        nodejs 'node'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        AWS_DEFAULT_REGION = 'ap-south-1'
        CLUSTER_NAME = 'poc-eks-cluster-1'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                    ${SCANNER_HOME}/bin/sonar-scanner \
                    -Dsonar.projectName=Hotstar \
                    -Dsonar.projectKey=Hotstar
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('OWASP Dependency Scan') {
            steps {
                dependencyCheck additionalArguments: '''
                    --scan ./ 
                    --disableYarnAudit 
                    --disableNodeAudit 
                    --nvdApiKey cec38a74-d5d5-4857-8180-7a0c5f36e6c8
                ''', odcInstallation: 'DC'

                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh '''
                        docker build -t 7760472594/hotstar:2 .
                        docker push 7760472594/hotstar:2
                        '''
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh 'trivy image 7760472594/hotstar:2 > trivyimage.txt'
            }
        }

        stage('Configure Kubeconfig for EKS') {
            steps {
                sh '''
                aws eks --region $AWS_DEFAULT_REGION update-kubeconfig --name $CLUSTER_NAME
                kubectl get nodes
                '''
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh 'kubectl apply -f K8S/'
            }
        }
    }

    post {
        always {
            emailext (
                subject: "Pipeline ${currentBuild.currentResult}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                <p>HOTSTAR CI/CD Pipeline Status</p>
                <p>Status: ${currentBuild.currentResult}</p>
                <p>Build URL: <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                """,
                to: 'shakthikumari7899@gmail.com',
                mimeType: 'text/html',
                attachmentsPattern: 'trivyimage.txt'
            )
        }
    }
}
