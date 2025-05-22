pipeline {
    agent any

    environment {
        VENV_DIR = 'venv'
        SONAR_SCANNER_HOME = tool 'SonarScanner'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    python3 -m venv ${VENV_DIR}
                    . ${VENV_DIR}/bin/activate
                    pip install --upgrade pip
                    if [ -f requirements.txt ]; then
                        pip install -r requirements.txt
                    else
                        echo "No requirements.txt found, creating minimal requirements"
                        echo "flake8==6.1.0" > requirements.txt
                        pip install -r requirements.txt
                    fi
                    pip install flake8
                '''
            }
        }

        

        stage('SAST - SonarQube Analysis') {
            environment {
                SONAR_TOKEN = credentials('Sonar-Token')
            }
            steps {
                withSonarQubeEnv('MySonarQube') {
                    sh '''
                        ${SONAR_SCANNER_HOME}/bin/sonar-scanner \
                          -Dsonar.projectKey=projet pfe \
                          -Dsonar.sources=. \
                          -Dsonar.host.url=$SONAR_HOST_URL \
                          -Dsonar.login=$SONAR_TOKEN \
                          -Dsonar.python.version=3.9 \
                          -Dsonar.python.coverage.reportPaths=coverage.xml \
                          -Dsonar.exclusions=venv/**,${VENV_DIR}/**
                    '''
                }
            }
        }

        stage('SCA - Dependency Check') {
            steps {
                sh '''
                    mkdir -p dependency-check-report
                    docker run --rm \
                      -v $(pwd):/src \
                      owasp/dependency-check \
                      --project "EDR App" \
                      --scan /src \
                      --format "HTML" \
                      --out /src/dependency-check-report \
                      --suppression /src/suppression.xml || true
                '''
            }
        }
        stage('Linting (flake8)') {
            steps {
                sh '''
                    . ${VENV_DIR}/bin/activate
                    mkdir -p reports
                    if [ -d app ]; then
                        flake8 app/ --statistics --output-file=reports/flake8.txt
                    else
                        # Corrigé: séparation claire entre find et xargs
                        find . -name "*.py" -print0 | xargs -0 flake8 --statistics --output-file=reports/flake8.txt || true
                    fi
                '''
            }
        }

        stage('Archive Reports') {
            steps {
                archiveArtifacts artifacts: 'dependency-check-report/*.html, reports/*.txt', fingerprint: true
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
