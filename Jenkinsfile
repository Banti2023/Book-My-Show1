pipeline {
    agent any

    environment {
        DOCKER_REGISTRY_CREDENTIALS = 'docker'
        SONAR_HOST = 'http://localhost:9000'   
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/Banti2023/Book-My-Show1.git'
                bat 'dir'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {   
                    script {
                        def scannerHome = tool 'sonar-scanner'   
                        withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_AUTH_TOKEN')]) {
                            bat """
                                "${scannerHome}\\bin\\sonar-scanner.bat" ^
                                -Dsonar.projectKey=bookmyshow ^
                                -Dsonar.projectName=BookMyShow ^
                                -Dsonar.login=${SONAR_AUTH_TOKEN} ^
                                -Dsonar.host.url=${SONAR_HOST} ^
                                -Dsonar.sources=bookmyshow-app ^
                                -Dsonar.exclusions=**/node_modules/**,**/dist/**,**/*.log
                            """
                        }
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Install NodeJS Dependencies') {
            steps {
                bat '''
                cd bookmyshow-app
                if exist package.json (
                    rmdir /s /q node_modules
                    del package-lock.json
                    npm install
                ) else (
                    echo package.json not found!
                    exit /b 1
                )
                '''
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: "${DOCKER_REGISTRY_CREDENTIALS}", toolName: 'docker') {
                        bat """
                        echo Building Docker image...
                        docker build --no-cache -t banti24/bookmyshow:latest -f bookmyshow-app\\Dockerfile bookmyshow-app

                        echo Pushing Docker image...
                        docker push banti24/bookmyshow:latest
                        """
                    }
                }
            }
        }

        stage('Deploy to Docker Container') {
            steps {
                bat """
                echo Stopping old container...
                docker stop bms || exit 0
                docker rm bms || exit 0

                echo Starting new container...
                docker run -d --restart=always --name bms -p 3000:3000 banti24/bookmyshow:latest

                docker ps -a
                docker logs bms
                """
            }
        }
    }

    post {
        success {
            emailext (
                subject: "SUCCESS: SonarQube Quality Gate Passed",
                body: """
                Hi Team,
                
                The SonarQube quality gate passed successfully.

                You can view the report here:
                http://44.227.100.103:9000/dashboard?id=Banti-app-code-quality
                username: admin, passwd: pass

                If you want to download a PDF report, please use the SonarQube interface or a PDF plugin.

                Regards,
                Jenkins
                """,
                to: 'bantithakuradarsh4@gmail.com'
            )
        }
        failure {
            emailext (
                subject: "FAILED: SonarQube Quality Gate",
                body: """
                Hi Team,
                
                The SonarQube quality gate has failed.
                
                Please check the details:
                http://44.227.100.103:9000/dashboard?id=Banti-app-code-quality
                username: admin, passwd: pass
                
                Regards,
                Jenkins
                """,
                to: 'bantithakuradarsh4@gmail.com'
            )
        }
        always {
            emailext attachLog: true,
                subject: "Build Result: '${currentBuild.result}'",
                body: "Project: ${env.JOB_NAME}<br/>" +
                      "Build Number: ${env.BUILD_NUMBER}<br/>" +
                      "URL: ${env.BUILD_URL}<br/>",
                to: 'bantithakuradarsh4@gmail.com'
        }
    }
}
