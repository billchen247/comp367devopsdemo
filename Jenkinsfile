pipeline {
    agent any
    triggers {
        cron('H/10 * * * 3')
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
        DOCKERHUB_REPO = "billchen247/comp367demorepo"
        IMAGE_NAME = "week7demo-app"
        IMAGE_TAG = "latest"
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
        
        stage('Package') {
            steps {
                echo "Packaging application..."
                sh 'mvn package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
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
            }
            steps {
                withSonarQubeEnv('LocalSonar') {
                    sh """
                       mvn sonar:sonar \
                       -Dsonar.projectKey=${env.SONAR_PROJECT_KEY}
                    """
                }
            }
            
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

       

        // ------------------------
        // MOCK CD SECTION
        // ------------------------

        stage('Build Docker Image') {
            when { branch 'master' }
            steps {
                echo "Building Docker image..."
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} . "
            }
        }

        stage('Push Docker Image in local registry') {
            when { branch 'master' }
            steps {
                echo "Mock pushing Docker image..."
                echo "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }

        stage('Docker Login') {
            when { branch 'master' }
            steps {
                withCredentials([string(credentialsId: 'dockerhubtoken', variable: 'DOCKERHUB_TOKEN')]) {
                    sh '''
                        docker login docker.io -u billchen247 -p $DOCKERHUB_TOKEN
                        
                    '''
                }
            }
        }

        stage('Tag for Docker Hub') {
            steps {
                sh """
                    echo "Tagging image for Docker Hub..."
                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} docker.io/${DOCKERHUB_REPO}/${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }

        stage('Push to Docker Hub') {
            steps {
                sh """
                    echo "Pushing image to Docker Hub..."
                    docker push docker.io/${DOCKERHUB_REPO}/${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }

        stage('Logout from Docker Hub') {
            steps {
                sh "docker logout docker.io"
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
