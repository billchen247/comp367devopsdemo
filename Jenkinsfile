pipeline {
    agent any
    triggers {
        cron('H/30 * * * 1')
    }

    tools {
        maven 'Maven'
        dockerTool 'Docker'
    }
    parameters {
        string(
            name: 'GITHUB_REPO_NAME',
            defaultValue: 'billchen247/JacocoExample',
            description: 'Enter the Github repo name to build'
        )
        string(
            name: 'BRANCH_NAME',
            defaultValue: 'master',
            description: 'Enter the Git branch to build'
        )
    }

    environment {
        APP_NAME = "week7demo-app"
        DOCKER_IMAGE = "myrepo/week7demo-app:latest"
        GITHUB_REPO_NAME = "${params.GITHUB_REPO_NAME}"
        BRANCH_NAME = "${params.BRANCH_NAME}"
    }

    stages {

        stage('Checkout') {
            steps {
                echo "fully working for this week demo"
                echo "Checking out source code..."
                echo "Checking out source code from branch: ${params.BRANCH_NAME} on github repo ${params.GITHUB_REPO_NAME}"

                git branch: "${params.BRANCH_NAME}",
                    url: "https://github.com/${params.GITHUB_REPO_NAME}"
            }
        }

        stage('Build') {
            steps {
                echo "Compiling source code..."
                sh 'mvn clean compile'
            }
        }

        stage('Unit Test') {
            steps {
                echo "Running unit tests..."
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Code Coverage (JaCoCo)') {
            steps {
                echo "Generating JaCoCo coverage report..."
                sh 'mvn jacoco:report'
            }
            
        }

        stage('Publish Coverage') {
            steps {
                jacoco(
                    execPattern: 'target/jacoco.exec',
                    classPattern: 'target/classes',
                    sourcePattern: 'src/main/java'
                )
            }
        }

        stage('Upload to GitHub Release') {
            steps {
                withCredentials([string(credentialsId: 'githubpat', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                        echo "Reading Maven project info..."
        
                        ARTIFACT_ID=$(mvn help:evaluate -Dexpression=project.artifactId -q -DforceStdout)
                        VERSION=$(mvn help:evaluate -Dexpression=project.version -q -DforceStdout)
        
                        JAR_FILE=target/${ARTIFACT_ID}-${VERSION}.jar
        
                        if [ ! -f "$JAR_FILE" ]; then
                            echo "ERROR: Artifact not found!"
                            exit 1
                        fi
        
                        echo "Using packaged artifact: $JAR_FILE"
        
                        echo "Creating GitHub Release..."
        
                        RESPONSE=$(curl -s -X POST \
                          -H "Authorization: token $GITHUB_TOKEN" \
                          -H "Accept: application/vnd.github+json" \
                          https://api.github.com/repos/${GITHUB_REPO_NAME}/releases \
                          -d "{\\"tag_name\\":\\"v$VERSION\\",\\"name\\":\\"v$VERSION\\",\\"generate_release_notes\\":true}")
        
                        RELEASE_ID=$(echo $RESPONSE | grep -o '"id":[0-9]*' | head -1 | grep -o '[0-9]*')
        
                        echo "Uploading artifact..."
        
                        curl -X POST \
                          -H "Authorization: token $GITHUB_TOKEN" \
                          -H "Content-Type: application/java-archive" \
                          --data-binary @$JAR_FILE \
                          "https://uploads.github.com/repos/${GITHUB_REPO_NAME}/releases/$RELEASE_ID/assets?name=$(basename $JAR_FILE)"
        
                        echo "Release completed successfully!"
                    '''
                }
            }
        }

        // gating part before CD
        stage('Gating before CD') {
            steps {
                echo "check some mockup gating before CD. like check change order approval or not"
            }
        }
        

        stage('Prepare Sonar Project Key') {
            steps {
                script {
                    // sanitize and convert github repo name to sonarqube project key
                    env.SONAR_PROJECT_KEY = params.GITHUB_REPO_NAME
                                                    .replaceAll('/', '_')
                                                    .replaceAll('[^a-zA-Z0-9_.-]', '')
                    echo "Sonar Project Key: ${env.SONAR_PROJECT_KEY}"
                }
            }
        }

        stage('SonarQube Analysis') {
            environment {
                scannerHome = tool 'SonarScanner'
                SONAR_TOKEN = credentials('sonar-token')
            }
            steps {
                withSonarQubeEnv('LocalSonar') {
                     sh """
                        ${scannerHome}/bin/sonar-scanner \
                          -Dsonar.host.url=http://host.docker.internal:19000 \
                          -Dsonar.login=${env.SONAR_TOKEN} \
                          -Dsonar.projectKey=${env.SONAR_PROJECT_KEY} \
                          -Dsonar.projectName="My Java Project" \
                          -Dsonar.projectVersion=1.0 \
                          -Dsonar.sources=src/main/java \
                          -Dsonar.tests=src/test/java \
                          -Dsonar.java.binaries=target/classes \
                          -Dsonar.java.test.binaries=target/test-classes \
                          -Dsonar.sourceEncoding=UTF-8 \
                          -Dsonar.language=java
                        """
                    
                    
                }
            }
            
        }

        stage('Check Quality Gate (Pull Model)') {
            steps {
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_AUTH_TOKEN')]) {
                    script {
    
                        // Read Sonar task info
                        // def props = readProperties file: 'target/sonar/report-task.txt' // only for maven sonar 
                        def props = readProperties file: '.scannerwork/report-task.txt' // for sonarscanner cli
    
                        def ceTaskId = props['ceTaskId']
                        def serverUrl = props['serverUrl']
    
                        echo "CE Task ID: ${ceTaskId}"
                        echo "Server URL: ${serverUrl}"
    
                        // Poll until analysis complete
                        timeout(time: 5, unit: 'MINUTES') {
                            waitUntil {
                                def response = sh(
                                    script: """
                                    curl -s -u ${SONAR_AUTH_TOKEN}: \
                                    ${serverUrl}/api/ce/task?id=${ceTaskId}
                                    """,
                                    returnStdout: true
                                ).trim()
    
                                def json = readJSON text: response
                                def status = json.task.status
    
                                echo "Current CE task status: ${status}"
    
                                if (status == "SUCCESS") {
                                    env.ANALYSIS_ID = json.task.analysisId
                                    return true
                                }
    
                                if (status == "FAILED" || status == "CANCELED") {
                                    error "SonarQube analysis failed"
                                }
    
                                sleep 5
                                return false
                            }
                        }
    
                        // Check Quality Gate
                        def qgResponse = sh(
                            script: """
                            curl -s -u ${SONAR_AUTH_TOKEN}: \
                            ${serverUrl}/api/qualitygates/project_status?analysisId=${env.ANALYSIS_ID}
                            """,
                            returnStdout: true
                        ).trim()
    
                        def qgJson = readJSON text: qgResponse
                        def qgStatus = qgJson.projectStatus.status
    
                        echo "Quality Gate Status: ${qgStatus}"
    
                        if (qgStatus != "OK") {
                            error "Pipeline failed due to Quality Gate: ${qgStatus}"
                        }
                    }
                }
            }
        }
        
        stage('Package') {
            steps {
                echo "Packaging application..."
                sh 'mvn package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }

        // ------------------------
        // MOCK CD SECTION
        // ------------------------

        stage('Build Docker Image') {
            when { branch 'master' }
            steps {
                echo "Building Docker image..."
                sh "docker build -t ${DOCKER_IMAGE} ."
            }
        }

        stage('Push Docker Image (Mock)') {
            when { branch 'master' }
            steps {
                echo "Mock pushing Docker image..."
                echo "docker push ${DOCKER_IMAGE}"
            }
        }

        stage('Deploy to Dev (Mock)') {
            when { branch 'master' }
            steps {
                echo "Mock deploy to Kubernetes..."
                echo "kubectl apply -f k8s/deployment.yaml"
            }
        }

        
        stage('start the application') {
            steps {
                sh '''
                    echo "demo to start the application by artifact package in local"
                    java -jar target/JacocoExample-0.0.1-SNAPSHOT.jar --server.port=8081
                '''
                
            }
        }

        stage('Cleanup') {
            steps {
                cleanWs()
            }
        }
    }
    

    
}
