pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }

    environment {
        SCANNER_HOME = tool 'SonarQube'
        DOCKER_IMAGE = 'omkarkadam0070/netflix'
        DOCKER_TAG   = "${BUILD_NUMBER}"
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/Omkar0070/netflix-devsecops.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=netflix-devsecops \
                        -Dsonar.projectName=netflix-devsecops \
                        -Dsonar.sources=src
                    """
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableNodeAudit --disableYarnAudit',
                                odcInstallation: 'Dependency-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Trivy File System Scan') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
        }

        stage('Docker Build & Push') {
            steps {
                withDockerRegistry(credentialsId: 'dockerhub') {
                    sh """
                        docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} .
                        docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh 'kubectl apply -f netflix-k8s/'
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh "trivy image ${DOCKER_IMAGE}:${DOCKER_TAG} > trivyimage.txt"
            }
        }
    }

    post {
        always {
            emailext(
                subject: "Build ${currentBuild.result}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                    <p><b>Status:</b> ${currentBuild.result}</p>
                    <p><b>Build:</b> #${env.BUILD_NUMBER}</p>
                    <p><b>URL:</b> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                """,
                attachmentsPattern: 'trivyfs.txt,trivyimage.txt',
                to: 'kadamom815@gmail.com'
            )
        }
    }
}
