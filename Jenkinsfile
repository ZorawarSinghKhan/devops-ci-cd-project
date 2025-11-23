pipeline {
    agent any

    environment {
        MVN = "${tool 'maven'}/bin/mvn"
        SONAR = "http://host.docker.internal:9000"
        NEXUS = "http://host.docker.internal:8081"
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Maven Build') {
            steps {
                sh "${MVN} clean package -DskipTests=false"
            }
        }

        stage('SonarQube Scan') {
            steps {
                withCredentials([string(credentialsId: 'sonar-token', variable: 'TOKEN')]) {
                    sh """
                       ${MVN} sonar:sonar \
                       -Dsonar.host.url=${SONAR} \
                       -Dsonar.login=$TOKEN
                    """
                }
            }
        }

        stage('Upload to Nexus') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'nexus-creds', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh """
                       ${MVN} deploy \
                       -DaltDeploymentRepository=internal::default::${NEXUS}/repository/maven-releases/ \
                       -Dnexus.username=$USER -Dnexus.password=$PASS
                    """
                }
            }
        }

        stage('Trivy Security Scan') {
            steps {
                sh "trivy filesystem --severity HIGH,CRITICAL --exit-code 0 ."
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    dockerImage = docker.build("yourdockerhubusername/devopsapp:latest")
                }

                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh """
                        echo "$PASS" | docker login -u "$USER" --password-stdin
                        docker push yourdockerhubusername/devopsapp:latest
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh "kubectl apply -f deployment.yaml"
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully!"
        }
        failure {
            echo "Pipeline failed. Check logs."
        }
    }
}

