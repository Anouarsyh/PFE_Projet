pipeline {
    agent any

    environment {
        VENV_DIR = 'venv'
        SONAR_SCANNER_HOME = tool 'SonarScanner'
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Récupération du code source...'
                checkout scm
            }
        }

        stage('Setup Python Environment') {
            steps {
                echo 'Configuration de l\'environnement Python...'
                sh '''
                    # Nettoyage de l'environnement précédent si existant
                    rm -rf ${VENV_DIR}
                    
                    # Création du nouvel environnement virtuel
                    python3 -m venv ${VENV_DIR}
                    . ${VENV_DIR}/bin/activate
                    
                    # Mise à jour de pip
                    pip install --upgrade pip
                    
                    # Installation des dépendances
                    if [ -f requirements.txt ]; then
                        echo "Installation des dépendances depuis requirements.txt"
                        pip install -r requirements.txt
                    else
                        echo "Aucun requirements.txt trouvé, création d'un fichier minimal"
                        echo "flake8==6.1.0" > requirements.txt
                        pip install -r requirements.txt
                    fi
                    
                    # Installation des outils de développement
                    pip install flake8 pytest coverage
                '''
            }
        }

        stage('Code Quality - Linting') {
            steps {
                echo 'Analyse de la qualité du code avec flake8...'
                sh '''
                    . ${VENV_DIR}/bin/activate
                    mkdir -p reports
                    
                    # Linting avec flake8
                    if [ -d app ]; then
                        echo "Analyse du répertoire app/"
                        flake8 app/ --statistics --output-file=reports/flake8.txt --exit-zero
                    else
                        echo "Analyse de tous les fichiers Python"
                        find . -name "*.py" -not -path "./${VENV_DIR}/*" | xargs flake8 --statistics --output-file=reports/flake8.txt --exit-zero || true
                    fi
                    
                    # Affichage du résumé
                    if [ -f reports/flake8.txt ]; then
                        echo "Résultats du linting :"
                        cat reports/flake8.txt
                    fi
                '''
            }
            post {
                always {
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'reports',
                        reportFiles: 'flake8.txt',
                        reportName: 'Flake8 Report'
                    ])
                }
            }
        }

        stage('SAST - SonarQube Analysis') {
            environment {
                SONAR_TOKEN = credentials('sonar-token')
            }
            steps {
                echo 'Analyse statique avec SonarQube...'
                withSonarQubeEnv('MySonarQube') {
                    sh '''
                        echo "Démarrage de l'analyse SonarQube"
                        ${SONAR_SCANNER_HOME}/bin/sonar-scanner \
                          -Dsonar.projectKey=Sonar-Token \
                          -Dsonar.projectName="projet pfe" \
                          -Dsonar.projectVersion=1.0 \
                          -Dsonar.sources=. \
                          -Dsonar.host.url=$SONAR_HOST_URL \
                          -Dsonar.login=$SONAR_TOKEN \
                          -Dsonar.python.version=3.9 \
                          -Dsonar.python.coverage.reportPaths=coverage.xml \
                          -Dsonar.exclusions=venv/**,${VENV_DIR}/**,**/__pycache__/**,*.pyc \
                          -Dsonar.python.flake8.reportPaths=reports/flake8.txt
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                echo 'Vérification du Quality Gate SonarQube...'
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('SCA - OWASP Dependency Check') {
            steps {
                echo 'Analyse des vulnérabilités des dépendances...'
                sh '''
                    # Création du répertoire de rapport
                    mkdir -p dependency-check-report
                    
                    # Vérification si le fichier de suppression existe
                    SUPPRESSION_ARG=""
                    if [ -f suppression.xml ]; then
                        echo "Fichier de suppression trouvé"
                        SUPPRESSION_ARG="--suppression /src/suppression.xml"
                    else
                        echo "Aucun fichier de suppression trouvé"
                    fi
                    
                    # Exécution de OWASP Dependency Check
                    docker run --rm \
                      -v $(pwd):/src \
                      -v dependency-check-data:/usr/share/dependency-check/data \
                      owasp/dependency-check \
                      --project "Projet PFE - EDR App" \
                      --scan /src \
                      --format "ALL" \
                      --out /src/dependency-check-report \
                      --enableRetired \
                      --enableExperimental \
                      $SUPPRESSION_ARG || true
                    
                    # Vérification des résultats
                    if [ -f dependency-check-report/dependency-check-report.html ]; then
                        echo "Rapport OWASP généré avec succès"
                    else
                        echo "Erreur lors de la génération du rapport OWASP"
                    fi
                '''
            }
            post {
                always {
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'dependency-check-report',
                        reportFiles: 'dependency-check-report.html',
                        reportName: 'OWASP Dependency Check Report'
                    ])
                }
            }
        }

        stage('Security Summary') {
            steps {
                echo 'Génération du résumé de sécurité...'
                sh '''
                    echo "=== RÉSUMÉ DE L'ANALYSE DE SÉCURITÉ ==="
                    echo "Date: $(date)"
                    echo "Projet: Projet PFE"
                    echo ""
                    
                    if [ -f reports/flake8.txt ]; then
                        echo "Linting (flake8): Terminé"
                        echo "Rapport disponible: reports/flake8.txt"
                    fi
                    
                    echo "Analyse SonarQube: Terminée"
                    echo "Vérifiez les résultats sur le serveur SonarQube"
                    
                    if [ -f dependency-check-report/dependency-check-report.html ]; then
                        echo "OWASP Dependency Check: Terminé"
                        echo "Rapport disponible: dependency-check-report/dependency-check-report.html"
                    fi
                    
                    echo "=== FIN DU RÉSUMÉ ==="
                '''
            }
        }

        stage('Archive Reports') {
            steps {
                echo 'Archivage des rapports...'
                archiveArtifacts artifacts: '''
                    dependency-check-report/*.html,
                    dependency-check-report/*.xml,
                    dependency-check-report/*.json,
                    reports/*.txt
                ''', fingerprint: true, allowEmptyArchive: true
            }
        }
    }

    post {
        always {
            echo 'Nettoyage de l\'espace de travail...'
            cleanWs()
        }
        success {
            echo 'Pipeline exécutée avec succès !'
        }
        failure {
            echo 'Échec de la pipeline. Vérifiez les logs.'
        }
        unstable {
            echo 'Pipeline instable. Certaines vérifications ont échoué.'
        }
    }
}
