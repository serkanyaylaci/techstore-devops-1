pipeline {
    agent any

    environment {
        DOCKER_IMAGE    = 'techstore-app'
        DOCKER_HUB_USER = 'serkanyaylaci'   // değiştir
        SONAR_HOST      = 'http://172.18.0.3:9000'
        SONAR_TOKEN     = credentials('sonar-token')
        SLACK_CHANNEL   = '#devops-techstore'
    }

    stages {

        // ─────────────────────────────
        stage('Checkout') {
            steps {
                checkout scm
                echo "✅ Kod alındı"
            }
        }

        // ─────────────────────────────
        stage('Setup') {
            steps {
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate

                    pip install --upgrade pip
                    pip install -r requirements.txt

                    echo "📦 Kurulum tamam"
                '''
            }
        }

        // ─────────────────────────────
        stage('Unit Tests') {
            steps {
                sh '''
                    . venv/bin/activate

                    # 🔥 KRİTİK FIX: app import hatası çözümü
                    export PYTHONPATH=$PWD

                    mkdir -p test-results

                    pytest tests/test_app.py -v \
                        --tb=short \
                        --junit-xml=test-results/unit-tests.xml \
                        --cov=app \
                        --cov-report=xml:coverage.xml \
                        --cov-report=term-missing
                '''
            }

            post {
                always {
                    junit 'test-results/unit-tests.xml'
                    publishCoverage adapters: [coberturaAdapter('coverage.xml')]
                }
            }
        }

        // ─────────────────────────────
        stage('SonarQube Analysis') {
    steps {
        withSonarQubeEnv('SonarQube') {
            script {
                def scannerHome = tool 'SonarQubeScanner'
                sh """
                    ${scannerHome}/bin/sonar-scanner \
                    -Dsonar.projectKey=TechStore \
                    -Dsonar.projectName=TechStore \
                    -Dsonar.sources=. \
                    -Dsonar.exclusions=venv/**,tests/** \
                    -Dsonar.python.coverage.reportPaths=coverage.xml \
                    -Dsonar.host.url=http://172.18.0.3:9000
                """
            }
        }
    }
}

        // ─────────────────────────────
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // ─────────────────────────────
        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t techstore-app:${BUILD_NUMBER} \
                                 -t techstore-app:latest .
                '''
            }
        }

        // ─────────────────────────────
        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-hub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin

                        docker tag techstore-app:latest $DOCKER_USER/techstore-app:${BUILD_NUMBER}
                        docker tag techstore-app:latest $DOCKER_USER/techstore-app:latest

                        docker push $DOCKER_USER/techstore-app:${BUILD_NUMBER}
                        docker push $DOCKER_USER/techstore-app:latest
                    '''
                }
            }
        }

        // ─────────────────────────────
        stage('Deploy') {
            steps {
                sh '''
                    docker stop techstore-app || true
                    docker rm techstore-app || true

                    docker run -d \
                        --name techstore-app \
                        -p 5000:5000 \
                        $DOCKER_HUB_USER/techstore-app:latest

                    echo "🚀 Deploy OK"
                '''
            }
        }

        // ─────────────────────────────
        stage('Smoke Test') {
            steps {
                sh '''
                    sleep 5

                    STATUS=$(curl -s -o /dev/null -w %{http_code} http://172.17.0.3:5000/health)

                    if [ "$STATUS" != "200" ]; then
                        echo "❌ Health failed: $STATUS"
                        exit 1
                    fi

                    echo "✅ Smoke test OK"
                '''
            }
        }

        // ─────────────────────────────
        stage('UI Tests') {
            steps {
                sh '''
                    . venv/bin/activate
                    export PYTHONPATH=$PWD

                    pytest tests/test_ui.py -v || true
                '''
            }
        }
    }

    // ─────────────────────────────
    post {
        success {
            echo "🎉 SUCCESS"

            slackSend(
                channel: env.SLACK_CHANNEL,
                color: 'good',
                message: """
✅ TechStore Deploy Başarılı
• Build: #${BUILD_NUMBER}
• Branch: ${BRANCH_NAME}
• Commit: ${GIT_COMMIT?.take(7)}
• URL: ${BUILD_URL}
                """
            )
        }

        failure {
            echo "❌ FAILED"

            slackSend(
                channel: env.SLACK_CHANNEL,
                color: 'danger',
                message: """
❌ TechStore Deploy FAIL
• Build: #${BUILD_NUMBER}
• Stage: ${STAGE_NAME}
• URL: ${BUILD_URL}console
                """
            )
        }

        always {
            sh "docker image prune -f || true"
            cleanWs()
        }
    }
}
