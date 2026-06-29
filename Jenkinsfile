pipeline {
    agent any

    tools {
        maven 'maven3'
        jdk 'jdk21'
    }

    environment {
        SCANNER_HOME = tool 'SonarScanner'
        SONARQUBE = credentials('sonar-token')

        AWS_REGION = 'us-east-1'
        AWS_ACCOUNT_ID = '822654906952'
        ECR_REPO_URL = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        ECR_APP_NAME = 'secureshop'
        IMAGE_REPO = "${ECR_REPO_URL}/${ECR_APP_NAME}"
        IMAGE_TAG = "${BUILD_NUMBER}"

        APP_NAME = 'SecureShop'
    }

    stages {

        stage('Git Checkout') {
            steps {
                checkout([$class: 'GitSCM',
                    branches: [[name: 'main']],
                    userRemoteConfigs: [[
                        url: 'https://github.com/meprana14-cell/SecureShop-DevSecOps.git',
                        credentialsId: 'GITPRIVATE_Key'
                    ]]
                ])
            }
        }

        stage('Compile Source Code') {
            steps {
                echo 'Compiling Source code...'
                sh 'mvn compile'
            }
        }

        stage('Unit Test') {
            steps {
                echo 'Running Unit Tests...'
                sh 'mvn test -DskipTests=true'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo 'Running SonarQube Code Analysis...'
                script {
                    withSonarQubeEnv('Sonar_Cube_Server') {
                        sh '''
                            ${SCANNER_HOME}/bin/sonar-scanner \
                                -Dsonar.projectKey=SecureShop \
                                -Dsonar.projectName=SecureShop \
                                -Dsonar.projectVersion=${BUILD_NUMBER} \
                                -Dsonar.sources=src \
                                -Dsonar.java.binaries=target/classes \
                                -Dsonar.sourceEncoding=UTF-8 \
                                -Dsonar.host.url=http://16.112.129.94:9000 \
                                -Dsonar.login=$SONARQUBE
                        '''
                    }
                }
            }
        }

        stage('Build Source Code') {
            steps {
                echo 'Packaging Source Code...'
                sh 'mvn package -DskipTests=true'
            }
        }

        stage('Artifact storing in Nexus') {
            steps {
                echo 'Publishing Artifact to Nexus Repository...'
                script {
                    withMaven(
                        globalMavenSettingsConfig: 'global-maven-settings',
                        jdk: 'jdk21',
                        maven: 'maven3',
                        traceability: true
                    ) {
                        sh 'mvn clean deploy -DskipTests=true'
                    }
                }
            }
        }

        stage('Build Container Image & Push to ECR') {
            steps {
                script {
                    IMAGE_TAG = "${BUILD_NUMBER ?: 'latest'}"
                    echo "Building Docker image: ${IMAGE_REPO}:${IMAGE_TAG}"

                    withAWS(region: "${AWS_REGION}", credentials: 'AWS_Credentials') {
                        sh """
                            aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO_URL}
                            docker build -t ${ECR_APP_NAME}:${IMAGE_TAG} .
                            docker tag ${ECR_APP_NAME}:${IMAGE_TAG} ${IMAGE_REPO}:${IMAGE_TAG}
                            docker push ${IMAGE_REPO}:${IMAGE_TAG}
                            docker image ls | grep ${ECR_APP_NAME}
                        """
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                script {
                    echo 'Scanning Docker image with Trivy...'
                    sh """
                        trivy image --exit-code 0 --severity HIGH,CRITICAL ${IMAGE_REPO}:${IMAGE_TAG} > trivy-report.txt
                        trivy image --exit-code 1 --severity CRITICAL ${IMAGE_REPO}:${IMAGE_TAG} >> trivy-report.txt || true
                    """
                    archiveArtifacts artifacts: 'trivy-report.txt', allowEmptyArchive: true
                    echo 'Trivy scan completed and report archived.'
                }
            }
        }

        stage('K8S Deploy') {
            steps {
                script {
                    def appName = 'secureshop'
                    def awsRegion = 'us-east-1'
                    def awsAccount = '822654906952'
                    def imageRepo = "${awsAccount}.dkr.ecr.${awsRegion}.amazonaws.com/${appName}"
                    def imageTag = "${BUILD_NUMBER ?: 'latest'}"

                    sh """
                        export APP_NAME=${appName}
                        export IMAGE_REPO=${imageRepo}
                        export IMAGE_TAG=${imageTag}

                        kubectl delete deployment secureshop -n secureapp --ignore-not-found
                        kubectl delete deployment secure-shop -n secureapp --ignore-not-found
                        kubectl delete svc secureshop-service -n secureapp --ignore-not-found
                        kubectl delete svc secure-shop-service -n secureapp --ignore-not-found

                        envsubst < kubernetes/deployment.yaml > /var/lib/jenkins/deployment-processed.yaml
                        envsubst < kubernetes/service.yaml > /var/lib/jenkins/service-processed.yaml

                        kubectl apply -f /var/lib/jenkins/deployment-processed.yaml
                        kubectl apply -f /var/lib/jenkins/service-processed.yaml

                        kubectl rollout status deployment/secureshop -n secureapp --timeout=120s
                        kubectl get svc -n secureapp
                    """
                }
            }
        }

        stage('Commit Version Update') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'GITHUB_Token', variable: 'GITHUB_TOKEN')]) {
                        sh 'git config user.email "meprana14@gmail.com"'
                        sh 'git config user.name "meprana14-cell"'
                        sh "git remote set-url origin https://$GITHUB_TOKEN:x-oauth-basic@github.com/meprana14-cell/SecureShop-DevSecOps.git"
                        sh 'git reset --hard'
                        sh 'git clean -fd'
                        sh 'git fetch origin main'
                        sh 'git pull --rebase origin main'
                        sh 'git add .'
                        sh 'git commit -m "ci: version bump [skip ci]" || echo "No changes to commit"'
                        sh 'git push origin HEAD:main'
                    }
                }
            }
        }
    }
}
