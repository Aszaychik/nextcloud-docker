pipeline {
    agent any
    environment {
        // Set your public IP (replace with actual IP)
        SERVER_IP = "157.15.212.5"
        
        // Port configuration (9000 is safe default)
        NEXTCLOUD_PORT = "9000"
        
        // Persistent storage location
        DATA_VOLUME = "/mnt/nextcloud-data"
        
        // Password credentials (set in Jenkins UI)
        DB_ROOT_PASS = credentials('nextcloud-db-root-pass')
        DB_PASS = credentials('nextcloud-db-pass')
        ADMIN_PASS = credentials('nextcloud-admin-pass')
    }
    stages {
        stage('Prepare Environment') {
            steps {
                sh """
                # Create storage directories
                sudo mkdir -p ${DATA_VOLUME}/{data,db,config}
                sudo chown -R 1000:1000 ${DATA_VOLUME}
                sudo chmod -R 775 ${DATA_VOLUME}
                
                # Generate env file
                mkdir -p deployments
                echo "MYSQL_ROOT_PASSWORD=${DB_ROOT_PASS}" > deployments/production.env
                echo "MYSQL_PASSWORD=${DB_PASS}" >> deployments/production.env
                echo "MYSQL_DATABASE=nextcloud" >> deployments/production.env
                echo "MYSQL_USER=nextcloud" >> deployments/production.env
                echo "NEXTCLOUD_ADMIN_USER=admin" >> deployments/production.env
                echo "NEXTCLOUD_ADMIN_PASSWORD=${ADMIN_PASS}" >> deployments/production.env
                echo "NEXTCLOUD_TRUSTED_DOMAINS=${SERVER_IP}" >> deployments/production.env
                """
            }
        }
        stage('Deploy Stack') {
            steps {
                sh 'docker compose -f docker-compose.yml down --remove-orphans || true'
                sh 'docker compose -f docker-compose.yml up -d --build'
            }
        }
        stage('Verify Deployment') {
            steps {
                sh "curl -I http://localhost:${NEXTCLOUD_PORT}"
            }
        }
    }
}