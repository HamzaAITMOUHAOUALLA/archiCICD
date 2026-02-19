pipeline {
    agent any

    environment {
        IMAGE_NAME = "petclinic-app:1.0"
        CONTAINER_NAME = "petclinic-container"
        STAGING_PORT = "8085"
    }

    stages {

        /* ================= CI ZONE ================= */

        stage('Checkout') {
            steps {
                git branch: 'master',
                    url: 'https://github.com/spring-petclinic/spring-petclinic-rest.git'
            }
        }


        stage('Build') {
            steps {
                sh 'chmod +x mvnw'
                sh './mvnw clean package -DskipTests'
            }
        }
        
        stage('Unit Tests') {
            steps {
                sh './mvnw test'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQubeServer') {
                    withCredentials([string(credentialsId: 'jenkinstoken', variable: 'SONAR_TOKEN')]) {
                        sh '''
                        ./mvnw sonar:sonar \
                        -Dsonar.login=$SONAR_TOKEN
                        '''
                    }
                }
            }
        }
        
        /* ================= SECURITY ZONE ================= */

        stage('Create Dockerfile') {
            steps {
                writeFile file: 'Dockerfile', text: '''
FROM eclipse-temurin:17-jdk-alpine
WORKDIR /app
COPY target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","app.jar"]
'''
            }
        }

        stage('Docker Build') {
            steps {
                sh "docker build -t ${IMAGE_NAME} ."
            }
        }
        
        stage('Trivy Security Scan') {
            steps {
                sh '''
                docker run --rm \
                -v /var/run/docker.sock:/var/run/docker.sock \
                aquasec/trivy:latest image \
                --severity HIGH,CRITICAL \
                --exit-code 0 \
                petclinic-app:1.0
                '''
            }
        }
        
        /* ================= STAGING ZONE ================= */

        stage('Clean Previous Container') {
            steps {
                sh '''
                docker stop ${CONTAINER_NAME} || true
                docker rm ${CONTAINER_NAME} || true
                '''
            }
        }

        stage('Deploy to Staging') {
        steps {
            sh '''
          
            
            docker run -d \
              --name ${CONTAINER_NAME} \
              --network ci-network \
              -p ${STAGING_PORT}:8080 \
              -e SERVER_PORT=8080 \
              -e SERVER_SERVLET_CONTEXT_PATH=/ \
              ${IMAGE_NAME}
            '''
        }
    }



       

    
        stage('STAGING E2E - Full CRUD Enterprise (REST Official)') { //CRUD (Create, Read, Update, Delete)
            steps {
                sh '''
                set -e
        
                echo "==========================================="
                echo "FULL CRUD ENTERPRISE - OFFICIAL REST API"
                echo "==========================================="
        
                # Health check (sans /api)
                HEALTH_URL="http://${CONTAINER_NAME}:8080/actuator/health"
        
                echo "Waiting for application health..."
                for i in $(seq 1 40); do
                  STATUS=$(curl -o /dev/null -s -w "%{http_code}" $HEALTH_URL || true)
                  if [ "$STATUS" = "200" ]; then
                    echo "Application Ready ✅"
                    break
                  fi
                  sleep 5
                done
        
                if [ "$STATUS" != "200" ]; then
                  echo "App not reachable ❌"
                  docker logs ${CONTAINER_NAME}
                  exit 1
                fi
        
                # Base URL pour CRUD API
                BASE_URL="http://${CONTAINER_NAME}:8080/api"
        
                ############################################
                # CREATE OWNER
                ############################################
                CREATE_RESPONSE=$(curl -s -w "\\n%{http_code}" \
                  -X POST $BASE_URL/owners \
                  -H "Content-Type: application/json" \
                  -u "user:fb92552c-4e8d-40c4-bee5-06aad6d4f9f6" \
                  -d '{
                    "firstName": "Enterprise",
                    "lastName": "Tester",
                    "address": "CI Street",
                    "city": "DevOps",
                    "telephone": "1234567890"
                  }')
        
                BODY=$(echo "$CREATE_RESPONSE" | head -n 1)
                STATUS=$(echo "$CREATE_RESPONSE" | tail -n 1)
        
                if [ "$STATUS" != "201" ]; then
                  echo "Create failed ❌ (HTTP $STATUS)"
                  exit 1
                fi
        
                NEW_ID=$(echo "$BODY" | jq -r '.id')
                echo "Created Owner ID: $NEW_ID ✅"
        
                ############################################
                # READ OWNER
                ############################################
                READ_JSON=$(curl -s -u "user:fb92552c-4e8d-40c4-bee5-06aad6d4f9f6" $BASE_URL/owners/$NEW_ID)
                echo "$READ_JSON" | jq -e '.firstName == "Enterprise"' > /dev/null || {
                  echo "Read validation failed ❌"
                  exit 1
                }
                echo "Read OK ✅"
        
                ############################################
                # UPDATE OWNER
                ############################################
                UPDATE_RESPONSE=$(curl -s -w "\\n%{http_code}" \
                  -X PUT $BASE_URL/owners/$NEW_ID \
                  -H "Content-Type: application/json" \
                  -u "user:fb92552c-4e8d-40c4-bee5-06aad6d4f9f6" \
                  -d '{
                    "id": '"$NEW_ID"',
                    "firstName": "Updated",
                    "lastName": "Tester",
                    "address": "CI Street",
                    "city": "DevOps",
                    "telephone": "1234567890"
                  }')
        
                STATUS=$(echo "$UPDATE_RESPONSE" | tail -n 1)
                if [ "$STATUS" != "204" ] && [ "$STATUS" != "200" ]; then
                  echo "Update failed ❌"
                  exit 1
                fi
        
                VERIFY_JSON=$(curl -s -u "user:fb92552c-4e8d-40c4-bee5-06aad6d4f9f6" $BASE_URL/owners/$NEW_ID)
                echo "$VERIFY_JSON" | jq -e '.firstName == "Updated"' > /dev/null || {
                  echo "Update validation failed ❌"
                  exit 1
                }
                echo "Update OK ✅"
        
                ############################################
                # DELETE OWNER
                ############################################
                DELETE_STATUS=$(curl -o /dev/null -s -w "%{http_code}" \
                  -X DELETE $BASE_URL/owners/$NEW_ID \
                  -u "user:fb92552c-4e8d-40c4-bee5-06aad6d4f9f6")
        
                if [ "$DELETE_STATUS" != "204" ]; then
                  echo "Delete failed ❌"
                  exit 1
                fi
                echo "Delete OK ✅"
        
                echo "==========================================="
                echo "FULL CRUD ENTERPRISE PASSED 🎉"
                echo "==========================================="
                '''
            }
        }





    }


    post {
        always {
            sh '''
            docker stop ${CONTAINER_NAME} || true
            docker rm ${CONTAINER_NAME} || true
            '''
        }
    }
}
