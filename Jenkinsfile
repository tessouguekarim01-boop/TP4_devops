
pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        disableConcurrentBuilds()
    }

    environment {
        CLIENT_IMAGE = 'mern-client'
        SERVER_IMAGE = 'mern-server'
        SERVER_CONTAINER = 'mern-server-test'
        CLIENT_CONTAINER = 'mern-client-test'
    }

    stages {

        stage('Checkout') {
            steps {
                deleteDir()
                checkout scm
            }
        }

        stage('Setup') {
            steps {
                bat 'node --version'
                bat 'npm --version'
                bat 'docker --version'

                dir('Client') {
                    retry(3) {
                        bat 'npm ci --fetch-retries=5 --fetch-retry-mintimeout=20000 --fetch-retry-maxtimeout=120000'
                    }
                }

                dir('Server') {
                    retry(3) {
                        bat 'npm ci --fetch-retries=5 --fetch-retry-mintimeout=20000 --fetch-retry-maxtimeout=120000'
                    }
                }
            }
        }

        stage('Build') {
            steps {
                dir('Client') {
                    bat 'npm run lint'
                    bat 'npm run build'
                }

                dir('Server') {
                    bat 'node --check src/index.mjs'
                }
            }
        }

        stage('Run Docker') {
            steps {
                script {
                    env.IMAGE_TAG = env.TAG_NAME?.trim() ? env.TAG_NAME : 'latest'

                    bat 'docker rm -f mern-server-test 2>NUL || exit /b 0'
                    bat 'docker rm -f mern-client-test 2>NUL || exit /b 0'

                    bat 'docker build -t mern-client:%IMAGE_TAG% ./Client'
                    bat 'docker build -t mern-server:%IMAGE_TAG% ./Server'

                    bat 'docker run -d --name mern-server-test -p 9000:9000 mern-server:%IMAGE_TAG%'
                    bat 'docker run -d --name mern-client-test -p 3000:3000 mern-client:%IMAGE_TAG%'
                }
            }
        }

        stage('Smoke Test') {
            steps {
                bat '''
                powershell -NoProfile -Command "$ok = $false; for ($i=0; $i -lt 15; $i++) { try { $r = Invoke-RestMethod http://localhost:9000/api/health -TimeoutSec 3; if ($r.message -eq 'API is running') { $ok = $true; break } } catch { Start-Sleep -Seconds 2 } }; if (-not $ok) { exit 1 }"
                '''

                bat '''
                powershell -NoProfile -Command "$r = Invoke-WebRequest http://localhost:3000 -UseBasicParsing; if ($r.StatusCode -ne 200) { exit 1 }"
                '''
            }
        }

        stage('Push DockerHub') {
            steps {
                script {
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'jenkins',
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_TOKEN'
                        )
                    ]) {
                        bat '''
                        powershell -NoProfile -NonInteractive -Command "$env:DOCKER_TOKEN | docker login --username $env:DOCKER_USER --password-stdin; if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }"
                        '''

                        bat 'docker tag mern-client:%IMAGE_TAG% %DOCKER_USER%/mern-client:%IMAGE_TAG%'
                        bat 'docker tag mern-server:%IMAGE_TAG% %DOCKER_USER%/mern-server:%IMAGE_TAG%'

                        bat 'docker push %DOCKER_USER%/mern-client:%IMAGE_TAG%'
                        bat 'docker push %DOCKER_USER%/mern-server:%IMAGE_TAG%'
                    }
                }
            }
        }

        stage('Archive') {
            steps {
                archiveArtifacts artifacts: 'Client/dist/**', fingerprint: true
            }
        }
    }

    post {
        always {
            bat 'docker rm -f mern-server-test 2>NUL || exit /b 0'
            bat 'docker rm -f mern-client-test 2>NUL || exit /b 0'
        }

        success {
            echo 'Pipeline completed successfully!'
        }

        failure {
            echo 'Pipeline failed! Check Console Output.'
        }
    }
}
