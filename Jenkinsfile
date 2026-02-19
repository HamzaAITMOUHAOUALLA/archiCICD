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
                git branch: 'main',
                    url: 'https://github.com/spring-projects/spring-petclinic.git'
            }
        }

        stage('Build') {
            steps {
                sh 'chmod +x mvnw'
                sh './mvnw clean package -DskipTests'
            }
        }
        /*
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
        */
        /* ================= SECURITY ZONE ================= */

        stage('Create Dockerfile') {
            steps {
                writeFile file: 'Dockerfile', text: '''
FROM eclipse-temurin:17-jdk-alpine
WORKDIR /app
COPY target/*.jar app.jar
ENTRYPOINT ["java","-jar","app.jar"]
'''
            }
        }

        stage('Docker Build') {
            steps {
                sh "docker build -t ${IMAGE_NAME} ."
            }
        }
        /*
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
        */
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
                ${IMAGE_NAME}
                '''
            }
        }

       

        stage('STAGING E2E - Full CRUD Enterprise (REST)') {
    steps {
        sh '''
        set -e

        echo "==========================================="
        echo "STAGING E2E - FULL CRUD ENTERPRISE (REST)"
        echo "==========================================="

        BASE_URL="http://${CONTAINER_NAME}:8080"

        ############################################
        # WAIT APPLICATION READY
        ############################################

        for i in $(seq 1 40); do
          STATUS=$(curl -o /dev/null -s -w "%{http_code}" \
            $BASE_URL/actuator/health || true)

          if [ "$STATUS" = "200" ]; then
            echo "Application Ready ✅"
            break
          fi

          echo "Waiting... ($i)"
          sleep 5
        done

        if [ "$STATUS" != "200" ]; then
          echo "Application not reachable ❌"
          docker logs ${CONTAINER_NAME}
          exit 1
        fi

        ############################################
        # VALIDATE VETS JSON STRUCTURE
        ############################################

        VETS_JSON=$(curl -s $BASE_URL/vets)

        echo "$VETS_JSON" | jq -e '.vetList | length > 0' > /dev/null || {
          echo "Vets JSON invalid ❌"
          exit 1
        }

        echo "Vets JSON structure OK ✅"

        ############################################
        # CREATE OWNER
        ############################################

        echo "Creating owner..."

        CREATE_RESPONSE=$(curl -s -w "\\n%{http_code}" \
          -X POST $BASE_URL/owners \
          -H "Content-Type: application/json" \
          -d '{
            "firstName": "Enterprise",
            "lastName": "Tester",
            "address": "CI Street",
            "city": "DevOps",
            "telephone": "1234567890"
          }')

        BODY=$(echo "$CREATE_RESPONSE" | head -n 1)
        STATUS=$(echo "$CREATE_RESPONSE" | tail -n 1)

        if [ "$STATUS" != "201" ] && [ "$STATUS" != "200" ]; then
          echo "Creation failed ❌ (HTTP $STATUS)"
          exit 1
        fi

        NEW_ID=$(echo "$BODY" | jq -r '.id')

        if [ -z "$NEW_ID" ] || [ "$NEW_ID" = "null" ]; then
          echo "Invalid ID ❌"
          exit 1
        fi

        echo "Owner created with ID $NEW_ID ✅"

        ############################################
        # READ OWNER
        ############################################

        READ_JSON=$(curl -s $BASE_URL/owners/$NEW_ID)

        FIRST_NAME=$(echo "$READ_JSON" | jq -r '.firstName')

        if [ "$FIRST_NAME" != "Enterprise" ]; then
          echo "Read verification failed ❌"
          exit 1
        fi

        echo "Read OK ✅"

        ############################################
        # UPDATE OWNER
        ############################################

        UPDATE_RESPONSE=$(curl -s -w "\\n%{http_code}" \
          -X PUT $BASE_URL/owners/$NEW_ID \
          -H "Content-Type: application/json" \
          -d '{
            "id": '"$NEW_ID"',
            "firstName": "Updated",
            "lastName": "Tester",
            "address": "CI Street",
            "city": "DevOps",
            "telephone": "1234567890"
          }')

        STATUS=$(echo "$UPDATE_RESPONSE" | tail -n 1)

        if [ "$STATUS" != "200" ]; then
          echo "Update failed ❌"
          exit 1
        fi

        VERIFY_JSON=$(curl -s $BASE_URL/owners/$NEW_ID)
        UPDATED_NAME=$(echo "$VERIFY_JSON" | jq -r '.firstName')

        if [ "$UPDATED_NAME" != "Updated" ]; then
          echo "Update verification failed ❌"
          exit 1
        fi

        echo "Update OK ✅"

        ############################################
        # DELETE OWNER
        ############################################

        DELETE_STATUS=$(curl -o /dev/null -s -w "%{http_code}" \
          -X DELETE $BASE_URL/owners/$NEW_ID)

        if [ "$DELETE_STATUS" != "204" ] && [ "$DELETE_STATUS" != "200" ]; then
          echo "Delete failed ❌"
          exit 1
        fi

        echo "Delete OK ✅"

        echo "==========================================="
        echo "FULL CRUD ENTERPRISE E2E PASSED 🎉"
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
