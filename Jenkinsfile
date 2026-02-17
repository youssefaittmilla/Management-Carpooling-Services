pipeline {
    agent any

    environment {
        SONARQUBE_ENV = 'sonar_integration'
        SONAR_PROJECT_KEY = 'Management-Carpooling-Services'
        SONAR_PROJECT_NAME = 'Management-Carpooling-Services'
        MAVEN_HOME = tool 'Maven'
        DOCKER_IMAGE = 'yassiramraoui/management-carpooling-services'
        DOCKER_TAG = 'latest'
    }

    stages {

        stage('Clone Repository') {
            steps {
                echo "📥 Cloning repository..."
                checkout scm
            }
        }

        stage('Compile Project') {
            steps {
                echo "🏗️ Compiling the code..."
                bat "\"${MAVEN_HOME}\\bin\\mvn\" clean compile"
            }
        }

        stage('Run Unit Tests') {
            steps {
                echo "🧪 Running tests..."
                bat "\"${MAVEN_HOME}\\bin\\mvn\" test"
            }
            post {
                always {
                    echo "📊 Publishing test results..."
                    junit allowEmptyResults: true, testResults: '**\\target\\surefire-reports\\*.xml'
                }
                success {
                    echo "✅ All tests passed!"
                }
                failure {
                    echo "❌ Tests failed! Check the test reports."
                }
            }
        }

        stage('Package Application') {
            steps {
                echo "📦 Packaging the application..."
                bat "\"${MAVEN_HOME}\\bin\\mvn\" package -DskipTests"
            }
            post {
                success {
                    archiveArtifacts artifacts: 'target\\*.jar, target\\*.war', fingerprint: true
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo "🔍 Running SonarQube analysis..."
                withSonarQubeEnv('sonar_integration') {
                    bat """
                        \"${MAVEN_HOME}\\bin\\mvn\" sonar:sonar ^
                        -Dsonar.projectKey=${SONAR_PROJECT_KEY} ^
                        -Dsonar.projectName=${SONAR_PROJECT_NAME} ^
                        -Dsonar.sources=src/main/java ^
                        -Dsonar.tests=src/test/java ^
                        -Dsonar.java.binaries=target/classes ^
                        -Dsonar.java.test.binaries=target/test-classes ^
                        -Dsonar.coverage.jacoco.xmlReportPaths=target\\site\\jacoco\\jacoco.xml ^
                        -Dsonar.junit.reportPaths=target\\surefire-reports
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                echo "🚦 Waiting for SonarQube Quality Gate..."
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('Docker Build') {
            steps {
                echo "🐳 Building Docker image..."
                bat "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} ."
            }
        }

        stage('Docker Push') {
            steps {
                echo "🚀 Pushing Docker image to Docker Hub..."
                withCredentials([usernamePassword(credentialsId: 'DockerHub', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                    bat "echo %DOCKER_PASSWORD% | docker login -u %DOCKER_USERNAME% --password-stdin"
                    bat "docker push ${DOCKER_IMAGE}:${DOCKER_TAG}"
                }
            }
        }

        stage('Docker Smoke Test') {
            steps {
                echo "🧪 Running container smoke test..."
                bat "docker run -d --rm -p 7070:7070 --name mcs_smoke ${DOCKER_IMAGE}:${DOCKER_TAG}"
                bat "timeout /t 5"
                bat "docker logs mcs_smoke | more"
                bat "docker rm -f mcs_smoke || exit 0"
            }
        }
    }

    post {
        always {
            echo "🧹 Cleaning workspace..."
            script {
                cleanWs()
            }
        }
        success {
            echo "✅ Pipeline finished successfully!"
        }
        failure {
            echo "❌ Pipeline failed. Check logs for errors."
        }
    }
}
