pipeline {
    agent any  // Führe die Pipeline auf einem beliebigen verfügbaren Agenten aus

    parameters {
        choice(
            name: 'TARGET_BRANCH',
            choices: ['none', 'main', 'dev'],  // 'none' für automatische Trigger
            description: 'Wähle die Branch aus, für die die Aktionen ausgeführt werden sollen (nur für manuelle Trigger).'
        )
    }

    environment {
        GIT_BRANCH = "${params.TARGET_BRANCH != 'none' ? params.TARGET_BRANCH : env.GIT_BRANCH}"
        EC2_USER = 'ubuntu'
        EC2_HOST = '35.159.66.99' // EC2-IP anpassen
        DEPLOY_PATH = '/var/www/nginx/react-app' // Nginx-Standardverzeichnis
        BACKEND_PATH = '/var/www/backend' // Verzeichnis für das Backend
        SSH_CREDENTIALS = 'jenkins-ec2-key'  
        APP_URL = "http://${EC2_HOST}"  // Dynamische URL basierend auf EC2_HOST
    }

    tools {
        nodejs 'NodeJS'
    }

    stages {
        stage('Check Branch') {
            steps {
                script {
                    // Überprüfe, welche Branch aktiv ist
                    if (env.GIT_BRANCH == 'origin/main' || env.GIT_BRANCH == 'main') {
                        echo "Push in die main-Branch erkannt oder manuell ausgewählt."
                        // Führe Aktionen für die main-Branch aus
                        stage('Main Branch Actions') {
                            echo "Starte Build für die main-Branch..."
                            sh 'echo "Führe Build-Schritte für die main-Branch aus..."'
                        }
                    } else if (env.GIT_BRANCH == 'origin/dev' || env.GIT_BRANCH == 'dev') {
                        echo "Push in die dev-Branch erkannt oder manuell ausgewählt."
                        // Führe Aktionen für die dev-Branch aus
                        stage('Dev Branch Actions') {
                            echo "Starte Build für die dev-Branch..."

                            // Clone Repository
                            stage('Clone Repository') {
                                steps {
                                    git branch: 'dev', 
                                        url: 'https://github.com/marcusBieber/react-build.git' // Repository für Deployment
                                }
                            }

                            // Führe Datenbank-Tests im Backend aus
                            stage('Run Database Tests') {
                                steps {
                                    dir('backend/database') {
                                        sh '''
                                            echo "Installiere Abhängigkeiten für Datenbank-Tests..."
                                            npm install
                                            echo "Führe Datenbank-Tests aus..."
                                            npm test
                                        '''
                                    }
                                }
                            }

                            // Installiere Abhängigkeiten und baue die React-App
                            stage('Install Dependencies and Build React App') {
                                steps {
                                    dir('frontend') {
                                        sh '''
                                            echo "Installiere Abhängigkeiten für das Frontend..."
                                            npm install
                                            echo "Baue die React-App..."
                                            npm run build
                                        '''
                                    }
                                }
                            }

                            // Kopiere das Backend auf die EC2-Instanz
                            stage('Deploy Backend to EC2') {
                                steps {
                                    script {
                                        sshagent(credentials: [SSH_CREDENTIALS]) {
                                            sh """
                                            echo "Erstelle Backend-Verzeichnis auf EC2..."
                                            ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} '
                                                sudo mkdir -p ${BACKEND_PATH} &&
                                                sudo chown -R ubuntu:ubuntu ${BACKEND_PATH} &&
                                                sudo rm -rf ${BACKEND_PATH}/*'
                                            
                                            echo "Kopiere Backend auf EC2..."
                                            scp -o StrictHostKeyChecking=no -r backend/* ${EC2_USER}@${EC2_HOST}:${BACKEND_PATH}

                                            echo "Installiere Abhängigkeiten und starte den Backend-Server..."
                                            ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} '
                                                cd ${BACKEND_PATH}/database &&
                                                npm install &&
                                                cd .. &&
                                                npm install &&
                                                if ! command -v pm2 &> /dev/null; then
                                                    echo "PM2 wird installiert..."
                                                    sudo npm install -g pm2
                                                fi &&
                                                pm2 start server.js --name "backend-server"'
                                            """
                                        }
                                    }
                                }
                            }

                            // Deploy Frontend to EC2
                            stage('Deploy Frontend to EC2') {
                                steps {
                                    script {
                                        sshagent(credentials: [SSH_CREDENTIALS]) {
                                            sh """
                                            ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} '
                                                echo "Prüfe Nginx-Installation..." &&
                                                if ! command -v nginx &> /dev/null; then
                                                    echo "Nginx wird installiert..." &&
                                                    sudo apt update &&
                                                    sudo apt install -y nginx &&
                                                    sudo systemctl enable nginx &&
                                                    sudo systemctl start nginx
                                                fi &&
                                                
                                                echo "Erstelle Deployment-Verzeichnis..." &&
                                                sudo mkdir -p ${DEPLOY_PATH} &&
                                                sudo chown -R ubuntu:ubuntu ${DEPLOY_PATH} &&
                                                sudo rm -rf ${DEPLOY_PATH}/* &&
                                                echo "Entferne Standard-Nginx-Site..." &&
                                                sudo rm -f /etc/nginx/sites-enabled/default
                                                '
                                            
                                            echo "Kopiere neuen Build auf EC2..."
                                            scp -o StrictHostKeyChecking=no -r frontend/dist/* ${EC2_USER}@${EC2_HOST}:${DEPLOY_PATH}

                                            ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} '
                                                sudo tee /etc/nginx/sites-available/react-app > /dev/null <<EOT
server {
    listen 80;
    server_name _;
    root ${DEPLOY_PATH};
    index index.html;
    
    location / {
        try_files \\\$uri /index.html;
    }

    error_page 404 /index.html;
}
EOT
                                                sudo ln -sf /etc/nginx/sites-available/react-app /etc/nginx/sites-enabled/react-app &&
                                                sudo systemctl restart nginx'
                                            """
                                        }
                                    }
                                }
                            }

                            // Check Website Availability
                            stage('Check Website Availability') {
                                steps {
                                    script {
                                        def response = sh(script: "curl -o /dev/null -s -w \"%{http_code}\" ${APP_URL}", returnStdout: true).trim()
                                        if (response != "200") {
                                            error("Website ist nicht erreichbar! HTTP-Status: ${response}")
                                        } else {
                                            echo "Website erfolgreich erreicht! HTTP-Status: ${response}"
                                        }
                                    }
                                }
                            }
                        }
                    } else {
                        echo "Unbekannte Branch erkannt oder ausgewählt: ${env.GIT_BRANCH}"
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline erfolgreich abgeschlossen!"
        }
        failure {
            echo "Pipeline fehlgeschlagen!"
        }
    }
}