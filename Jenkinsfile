pipeline {
    agent any
    environment {
        SERVER_IP = "157.15.212.5"
        NEXTCLOUD_PORT = "9000"
        DATA_VOLUME = "/mnt/nextcloud-data"
        ENV_FILE = "${WORKSPACE}/deployments/production.env"
    }
    stages {
        stage('Prepare Environment') {
            steps {
                withCredentials([file(credentialsId: 'nextcloud-env-file', variable: 'SECRET_ENV')]) {
                    sh """
                    mkdir -p ${WORKSPACE}/deployments
                    sudo mkdir -p ${DATA_VOLUME}/{data,db,config}
                    
                    # Copy and customize env file
                    cp '$SECRET_ENV' '${ENV_FILE}'
                    sed -i "s/YOUR_PUBLIC_IP/${SERVER_IP}/g" '${ENV_FILE}'
                    
                    # CORRECT PERMISSIONS (UID 33 is www-data)
                    sudo chown -R 33:33 ${DATA_VOLUME}/data
                    sudo chown -R 33:33 ${DATA_VOLUME}/config
                    sudo chown -R 999:999 ${DATA_VOLUME}/db  # For MariaDB
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
                sh """
                # Wait for containers to become healthy
                echo "Waiting for containers to become healthy..."
                timeout 120s bash -c "while ! docker inspect nextcloud-db --format '{{.State.Health.Status}}' | grep -q 'healthy'; do sleep 5; done"
                timeout 120s bash -c "while ! docker inspect nextcloud-app --format '{{.State.Health.Status}}' | grep -q 'healthy'; do sleep 5; done"
                
                # Check container status
                echo "\\n\\n=== Container status ==="
                docker ps -a --format "table {{.Names}}\\t{{.Status}}\\t{{.Ports}}"
                
                # Check container logs
                echo "\\n\\n=== Nextcloud logs ==="
                docker logs nextcloud-app --tail 100
                echo "\\n\\n=== Database logs ==="
                docker logs nextcloud-db --tail 50
                
                # Check host port binding
                echo "\\n\\n=== Port bindings ==="
                docker port nextcloud-app
                
                # Check server listening ports
                echo "\\n\\n=== Listening ports ==="
                sudo netstat -tulpn | grep ':${NEXTCLOUD_PORT}'
                
                # Test local access
                echo "\\n\\n=== Local access test ==="
                curl -I http://localhost:${NEXTCLOUD_PORT} || echo "Local access failed"
                
                # Test external access
                echo "\\n\\n=== External access test ==="
                curl -I --connect-timeout 5 http://${SERVER_IP}:${NEXTCLOUD_PORT} || echo "External access test failed"
                """
            }
        }
    }
    post {
        always {
            sh "rm -f '${ENV_FILE}' || true"
        }
    }
}