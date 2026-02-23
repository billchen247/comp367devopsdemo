pipeline {
    agent any

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
        APP_NAME = "week6demo-app"
        DOCKER_IMAGE = "myrepo/week6demo-app:latest"
    }

    stages {

        stage('Checkout') {
            steps {
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
            }
            steps {
                withSonarQubeEnv('LocalSonar') {
                    sh """
                       mvn sonar:sonar 
                       -Dsonar.projectKey=${env.SONAR_PROJECT_KEY} \
                       -Dsonar.host.url=http://localhost:19000 \
                       -Dsonar.login=${sonar-token}
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
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

        stage('Cleanup') {
            steps {
                cleanWs()
            }
        }
    }
    

    
}
