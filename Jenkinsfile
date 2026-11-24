
pipeline {
    agent any

    environment {
        CLIENT_IMAGE = 'mern-client'
        SERVER_IMAGE = 'mern-server'
        SERVER_CONTAINER = 'mern-server-test'
        CLIENT_CONTAINER = 'mern-client-test'
    }

    stages {

        // 1. CHECKOUT
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        // 2. SETUP
        stage('Setup') {
            steps {
                bat 'node --version'
                bat 'npm --version'
                bat 'docker --version'

                dir('Client') {
                    bat 'npm ci'
                }

                dir('Server') {
                    bat 'npm ci'
                }
            }
        }

        // 3. BUILD
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

        // 4. RUN DOCKER
        stage('Run Docker') {
            steps {
                script {
                    def version = env.TAG_NAME ?: 'latest'

                    bat 'docker build -t ' + CLIENT_IMAGE + ':' + version + ' ./Client'
                    bat 'docker build -t ' + SERVER_IMAGE + ':' + version + ' ./Server'

                    bat 'docker rm -f ' + SERVER_CONTAINER + ' 2>NUL || exit /b 0'
                    bat 'docker rm -f ' + CLIENT_CONTAINER + ' 2>NUL || exit /b 0'

                    bat 'docker run -d --name ' + SERVER_CONTAINER + ' -p 9000:9000 ' + SERVER_IMAGE + ':' + version
                    bat 'docker run -d --name ' + CLIENT_CONTAINER + ' -p 3000:3000 ' + CLIENT_IMAGE + ':' + version
                }
            }
        }

        // 5. SMOKE TEST
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

        // 6. ARCHIVE
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
            echo 'CI/CD Pipeline completed successfully!'
        }

        failure {
            echo 'CI/CD Pipeline failed. Check Console Output.'
        }
    }
}
