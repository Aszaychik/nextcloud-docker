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
                script {
                    // Create base directories first
                    sh """
                    echo "Creating workspace deployments directory"
                    mkdir -p ${WORKSPACE}/deployments
                    
                    echo "Creating persistent storage directories"
                    sudo mkdir -p ${DATA_VOLUME}
                    sudo mkdir -p ${DATA_VOLUME}/data
                    sudo mkdir -p ${DATA_VOLUME}/db
                    sudo mkdir -p ${DATA_VOLUME}/config
                    
                    echo "Setting initial permissions"
                    sudo chmod 775 ${DATA_VOLUME}
                    """
                    
                    // Process credentials securely
                    withCredentials([file(credentialsId: 'nextcloud-env-file', variable: 'SECRET_ENV')]) {
                        // Use sh step with script parameter for secure handling
                        sh script: """
                        echo "Processing environment file"
                        cp '${SECRET_ENV}' '${ENV_FILE}'
                        sed -i "s/YOUR_PUBLIC_IP/${SERVER_IP}/g" '${ENV_FILE}'
                        chmod 600 '${ENV_FILE}'
                        """, label: "Process environment file"
                    }
                    
                    // Set final directory permissions
                    sh """
                    echo "Setting application-specific permissions"
                    sudo chown -R 33:33 ${DATA_VOLUME}/data
                    sudo chown -R 33:33 ${DATA_VOLUME}/config
                    sudo chown -R 999:999 ${DATA_VOLUME}/db
                    sudo chmod -R 775 ${DATA_VOLUME}
                    
                    echo "Verifying directory structure:"
                    ls -ld ${DATA_VOLUME}
                    ls -l ${DATA_VOLUME}
                    """
                }
            }
        }
        
        stage('Deploy Stack') {
            steps {
                sh """
                echo "Stopping any existing containers"
                docker compose -f docker-compose.yml down --remove-orphans || true
                
                echo "Starting new containers"
                docker compose -f docker-compose.yml up -d --build
                
                echo "Container status:"
                docker ps -a
                """
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    // Simple check if containers exist
                    def containers = sh(script: "docker ps -a --format '{{.Names}}'", returnStdout: true).trim()
                    
                    if (containers.contains("nextcloud-db") && containers.contains("nextcloud-app")) {
                        echo "Containers created successfully"
                        
                        // Wait for services to be ready
                        sh """
                        # Wait for database
                        echo "Waiting for database to become ready..."
                        timeout 120s bash -c "until docker exec nextcloud-db mysqladmin ping -uroot -p'@tebalocentral' --silent; do sleep 5; echo 'Waiting...'; done"
                        
                        # Wait for Nextcloud
                        echo "Waiting for Nextcloud to become ready..."
                        timeout 180s bash -c "until docker exec nextcloud-app curl -sIf http://localhost >/dev/null; do sleep 5; echo 'Waiting...'; done"
                        """
                        
                        // Perform verification checks
                        sh """
                        # Check container status
                        echo "\\n\\n=== Container status ==="
                        docker ps -a --format "table {{.Names}}\\t{{.Status}}\\t{{.Ports}}"
                        
                        # Check container logs
                        echo "\\n\\n=== Nextcloud logs (last 50 lines) ==="
                        docker logs nextcloud-app --tail 50
                        
                        # Check host port binding
                        echo "\\n\\n=== Port bindings ==="
                        docker port nextcloud-app
                        
                        # Test local access
                        echo "\\n\\n=== Local access test ==="
                        curl -I http://localhost:${NEXTCLOUD_PORT} || echo "Local access failed"
                        """
                        
                        // External access test with error handling
                        def externalAccess = sh(
                            script: "curl -Is --connect-timeout 5 http://${SERVER_IP}:${NEXTCLOUD_PORT}",
                            returnStatus: true
                        )
                        echo "External access test returned status: ${externalAccess}"
                    } else {
                        echo "Containers not created. Skipping verification."
                    }
                }
            }
        }
    }
    
    post {
        always {
            sh "rm -f '${ENV_FILE}' || true"
            
            // Capture logs for debugging
            script {
                try {
                    sh "docker logs nextcloud-app > nextcloud-app.log 2>&1 || true"
                    sh "docker logs nextcloud-db > nextcloud-db.log 2>&1 || true"
                    archiveArtifacts artifacts: '*.log', allowEmptyArchive: true
                } catch (Exception e) {
                    echo "Error capturing logs: ${e}"
                }
            }
        }
    }
}