pipeline {
    agent any

    tools {
        jdk 'jdk21'
        maven 'maven'
    }

    environment {
        SCANNER_HOME = tool 'sonarqube-scanner'
        NEXUS_VERSION = "nexus3"
        NEXUS_URL = "3.6.34.143:8081"
        NEXUS_PROTOCOL = "http"
        NEXUS_REPOSITORY = "Boardgame"
        NEXUS_CREDENTIAL_ID = 'nexus-creds'
        IMAGE_NAME = "boardgame-image"
        AWS_REGION = "ap-south-1"
        ECR_ACCOUNT_ID = "867344449786"
        ECR_REPO = "workshop/boardgame"
        EKS_CLUSTER = "Workshop-Demo-Cluster"
        ECR_REGISTR = "867344449786.dkr.ecr.ap-south-1.amazonaws.com"
    }

    stages {
        // stage('Checkout') {
        //     steps {
        //         git branch: 'main', url: 'https://github.com/Preet814/Boardgame.git'
        //     }
        // }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube-server') {  
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectKey=Boardgame \
                        -Dsonar.sources=src \
                        -Dsonar.java.binaries=target
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean install'
            }
        }

        stage("Publish to Nexus") {
            steps {
                script {
                    pom = readMavenPom file: "pom.xml"
                    filesByGlob = findFiles(glob: "target/*.${pom.packaging}")
                    artifactPath = filesByGlob[0].path
                    artifactExists = fileExists artifactPath
                    if (artifactExists) {
                        nexusArtifactUploader(
                            nexusVersion: NEXUS_VERSION,
                            protocol: NEXUS_PROTOCOL,
                            nexusUrl: NEXUS_URL,
                            groupId: pom.groupId,
                            version: pom.version,
                            repository: NEXUS_REPOSITORY,
                            credentialsId: NEXUS_CREDENTIAL_ID,
                            artifacts: [
                                [artifactId: pom.artifactId,
                                classifier: '',
                                file: artifactPath,
                                type: pom.packaging]
                            ]
                        )
                    } else {
                        error "*** File: ${artifactPath}, could not be found"
                    }
                }
            }
        }

        stage('Docker Build & Trivy Scan') {
            steps {
                script {
                    sh """
                        # Build Docker image
                        docker build -t ${IMAGE_NAME}:${env.BUILD_NUMBER} .

                        # Run Trivy scan (report only, do not fail pipeline)
                        trivy image --severity HIGH,CRITICAL ${IMAGE_NAME}:${env.BUILD_NUMBER} > trivy-report.txt || true
                    """
                }
            }
        }

        stage('Push to ECR') {
            steps {
                script {
                    sh """
                        # Tag image with Jenkins BUILD_ID
                        docker tag ${IMAGE_NAME}:${env.BUILD_NUMBER} ${ECR_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${env.BUILD_NUMBER}
                        
                        # Login to ECR using EC2 IAM role
                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                        
                        # Push image
                        docker push ${ECR_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${env.BUILD_NUMBER}
                    """
                }
            }
            post {
                success {
                    sh """
                        docker rmi ${IMAGE_NAME}:${env.BUILD_NUMBER} || true
                    """
                }
            }
        }
        stage('Deploy to EKS') {
            steps {
                script {
                    sh """
                       sed -e "s|ECR_REGISTRY|${ECR_REGISTR}|g" \
                           -e "s|ECR_REPO|${ECR_REPO}|g" \
                           -e "s|BUILD_ID|${env.BUILD_NUMBER}|g" \
                           deployment-service.yaml > deployment.yaml
                    """
                }
                script {
                    sh 'cat deployment.yaml'
                    sh "aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER}"
                    sh 'kubectl apply -f deployment.yaml'
                }
            }
        }
    }
    post {
        failure {
            script {
                def trivyReport = fileExists('trivy-report.txt') ? readFile('trivy-report.txt') : 'No Trivy report generated.'
                mail to: 'preetmundra814@gmail.com',
                    subject: "❌ FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    body: """Build failed for job: ${env.JOB_NAME}
    Build number: ${env.BUILD_NUMBER}

    Check logs: ${env.BUILD_URL}
    Blue Ocean: ${env.RUN_DISPLAY_URL}

    Trivy Report:
    ${trivyReport}
    """
            }
        }
        success {
            script {
                def trivyReport = fileExists('trivy-report.txt') ? readFile('trivy-report.txt') : 'No Trivy report generated.'
                mail to: 'preetmundra814@gmail.com',
                    subject: "✅ SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    body: """Build successful for job: ${env.JOB_NAME}
    Build number: ${env.BUILD_NUMBER}

    Check details: ${env.BUILD_URL}
    Blue Ocean: ${env.RUN_DISPLAY_URL}

    Trivy Report:
    ${trivyReport}
    """
            }
        }
    }
}
