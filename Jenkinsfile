pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_IMAGE = 'ayushdocker2607/netflix'
        DOCKER_TAG = "${BUILD_NUMBER}"
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'newbranch', url: 'https://github.com/Ayush-bhoyar/Netflix-DevSecOps-Project.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectName=Netflix \
                        -Dsonar.projectKey=Netflix'''
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('OWASP FS Scan') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'Docker-cred', toolName: 'docker') {
                        sh """
                            docker build --build-arg TMDB_V3_API_KEY=9a2c05102991f80c0113ca9bc884cec3 \
                            -t ${DOCKER_IMAGE}:${DOCKER_TAG} .
                            docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                        """
                    }
                }
            }
        }

        stage('Update Kubernetes Manifest') {
            steps {
                script {
                    // Update image tag in deployment.yaml using regex
                    sh """
                        sed -i 's|image: ${DOCKER_IMAGE}:.*|image: ${DOCKER_IMAGE}:${DOCKER_TAG}|' Kubernetes/deployment.yml
                    """
                }
            }
        }

        stage('Push Updated Manifest to GitHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github-cred', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                    script {
                        sh """
                            git config user.name "Jenkins CI"
                            git config user.email "jenkins@example.com"
                            git add Kubernetes/deployment.yml || true
                            git commit -m "Update image tag to ${DOCKER_TAG}" || true
                            git push https://${GIT_USER}:${GIT_PASS}@github.com/Ayush-bhoyar/Netflix-DevSecOps-Project.git newbranch || true
                        """
                    }
                }
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
                attachLog: true,
                subject: "Build ${currentBuild.result}: ${env.JOB_NAME} [#${env.BUILD_NUMBER}]",
                body: """
                    <p><b>Build Result:</b> ${currentBuild.result}</p>
                    <p><b>Project:</b> ${env.JOB_NAME}</p>
                    <p><b>Build Number:</b> ${env.BUILD_NUMBER}</p>
                    <p><b>Build URL:</b> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                """,
                to: 'bhoyarayush71@gmail.com',
                attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
            )
        }
    }
}
