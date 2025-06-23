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
                    // Create directories first
                    sh """
                    mkdir -p ${WORKSPACE}/deployments
                    sudo mkdir -p ${DATA_VOLUME}/{data,db,config}
                    """
                    
                    // Process credentials securely
                    withCredentials([file(credentialsId: 'nextcloud-env-file', variable: 'SECRET_ENV')]) {
                        // Use sh step with script parameter for secure handling
                        sh script: """
                        # Copy and customize env file
                        cp '${SECRET_ENV}' '${ENV_FILE}'
                        sed -i "s/YOUR_PUBLIC_IP/${SERVER_IP}/g" '${ENV_FILE}'
                        
                        # Set permissions
                        chmod 600 '${ENV_FILE}'
                        """, label: "Process environment file"
                    }
                    
                    // Set directory permissions
                    sh """
                    sudo chown -R 33:33 ${DATA_VOLUME}/data
                    sudo chown -R 33:33 ${DATA_VOLUME}/config
                    sudo chown -R 999:999 ${DATA_VOLUME}/db
                    sudo chmod -R 775 ${DATA_VOLUME}
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
                script {
                    // Check if containers are running
                    def dbRunning = sh(script: "docker inspect -f '{{.State.Running}}' nextcloud-db", returnStdout: true).trim()
                    def appRunning = sh(script: "docker inspect -f '{{.State.Running}}' nextcloud-app", returnStdout: true).trim()
                    
                    if (dbRunning != "true" || appRunning != "true") {
                        error("Containers are not running. Aborting verification.")
                    }
                    
                    // Wait for services to be ready
                    sh """
                    # Wait for database
                    timeout 120s bash -c "until docker exec nextcloud-db mysqladmin ping -uroot -p'@tebalocentral' --silent; do sleep 5; echo 'Waiting for database...'; done"
                    
                    # Wait for Nextcloud
                    timeout 180s bash -c "until docker exec nextcloud-app curl -sIf http://localhost >/dev/null; do sleep 5; echo 'Waiting for Nextcloud...'; done"
                    """
                    
                    // Perform verification checks
                    sh """
                    # Check container status
                    echo "\\n\\n=== Container status ==="
                    docker ps -a --format "table {{.Names}}\\t{{.Status}}\\t{{.Ports}}"
                    
                    # Check container logs
                    echo "\\n\\n=== Nextcloud logs ==="
                    docker logs nextcloud-app --tail 100
                    
                    # Check host port binding
                    echo "\\n\\n=== Port bindings ==="
                    docker port nextcloud-app
                    
                    # Check server listening ports
                    echo "\\n\\n=== Listening ports ==="
                    sudo netstat -tulpn | grep ':${NEXTCLOUD_PORT}' || true
                    
                    # Test local access
                    echo "\\n\\n=== Local access test ==="
                    curl -I http://localhost:${NEXTCLOUD_PORT} || echo "Local access failed"
                    """
                    
                    // External access test with error handling
                    def externalAccess = sh(
                        script: "curl -Is --connect-timeout 5 http://${SERVER_IP}:${NEXTCLOUD_PORT}",
                        returnStatus: true
                    )
                    echo "External access test returned: ${externalAccess}"
                }
            }
        }
    }
    post {
        always {
            sh "rm -f '${ENV_FILE}' || true"
            
            // Capture logs for debugging
            script {
                sh "docker logs nextcloud-app > nextcloud-app.log 2>&1 || true"
                sh "docker logs nextcloud-db > nextcloud-db.log 2>&1 || true"
                archiveArtifacts artifacts: '*.log', allowEmptyArchive: true
            }
        }
    }
}