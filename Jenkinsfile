pipeline {
    agent any
    environment {
        SERVER_IP = "157.15.212.5" // Replace with your server's public IP
        NEXTCLOUD_PORT = "9000"
        DATA_VOLUME = "/mnt/nextcloud-data"
        ENV_FILE = "${WORKSPACE}/deployments/production.env"
    }
    stages {
        stage('Prepare Environment') {
            steps {
                // Load secret file credential
                withCredentials([file(credentialsId: 'nextcloud-env-file', variable: 'SECRET_ENV')]) {
                    sh """
                    # Create directories
                    mkdir -p ${WORKSPACE}/deployments
                    sudo mkdir -p ${DATA_VOLUME}/{data,db,config}
                    
                    # Copy secret file to workspace
                    cp '$SECRET_ENV' '${ENV_FILE}'
                    
                    # Replace placeholder with actual IP
                    sed -i "s/YOUR_PUBLIC_IP/${SERVER_IP}/g" '${ENV_FILE}'
                    
                    # Set permissions
                    sudo chown -R 1000:1000 ${DATA_VOLUME}
                    sudo chmod -R 775 ${DATA_VOLUME}
                    chmod 600 '${ENV_FILE}'
                    """
                }
            }
        }
        
        stage('Deploy Stack') {
            steps {
                sh """
                docker compose -f docker-compose.yml down --remove-orphans || true
                docker compose -f docker-compose.yml up -d --build
                """
            }
        }
        
        stage('Verify Deployment') {
            steps {
                sh "sleep 30"  // Wait for containers to initialize
                sh "curl -I http://${SERVER_IP}:${NEXTCLOUD_PORT}"
            }
        }
    }
    
    post {
        always {
            // Securely clean up the environment file
            sh "rm -f '${ENV_FILE}' || true"
        }
    }
}