pipeline {
    agent any

    tools {
        jdk 'jdk'
        nodejs 'node17'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {

        stage('clean workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'main',
                    credentialsId: 'github-token',
                    url: 'https://github.com/nikhilmalik0399/Devops-Startbucks.git'
            }
        }

        stage('Sonarqube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                    $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=starbucks \
                    -Dsonar.projectKey=starbucks
                    '''
                }
            }
        }

        stage('quality gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('TRIVY FS SCAN') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker') {
                        sh 'docker build -t starbucks .'
                        sh 'docker tag starbucks nikhilmalik99/starbucks:latest'
                        sh 'docker push nikhilmalik99/starbucks:latest'
                    }
                }
            }
        }

        stage('TRIVY') {
            steps {
                sh 'trivy image nikhilmalik99/starbucks:latest > trivyimage.txt'
            }
        }

        stage('App Deploy to Docker container') {
            steps {
                sh '''
                docker rm -f starbucks || true
                docker run -d --name starbucks -p 3000:3000 nikhilmalik99/starbucks:latest
                '''
            }
        }

        stage('Deploy to EKS Cluster') {
            steps {
        dir('kubernetes') {
            withAWS(credentials: 'aws-creds', region: 'ap-south-1') {
                sh '''
                aws sts get-caller-identity
                aws eks update-kubeconfig --region ap-south-1 --name Cloudaseem
                kubectl apply -f manifest.yml
                kubectl get pods
                kubectl get svc
                '''
            }
        }
    }
        }
    }

    post {
        always {
            script {
                def buildStatus = currentBuild.currentResult
                def buildUser = currentBuild.getBuildCauses('hudson.model.Cause$UserIdCause')[0]?.userId ?: 'Github User'

                emailext(
                    subject: "Pipeline ${buildStatus}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    body: """
                        <p>This is a Jenkins Starbucks CI/CD pipeline status.</p>
                        <p>Project: ${env.JOB_NAME}</p>
                        <p>Build Number: ${env.BUILD_NUMBER}</p>
                        <p>Build Status: ${buildStatus}</p>
                        <p>Started by: ${buildUser}</p>
                        <p>Build URL: <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                    """,
                    to: 'nikhilmalik0399@gmail.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
                )
            }
        }
    }
}
