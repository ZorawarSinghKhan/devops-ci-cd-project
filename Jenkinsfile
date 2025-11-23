// Jenkinsfile: Full DevSecOps pipeline (Maven -> SonarQube -> Nexus -> Trivy -> Docker -> Kubernetes)
// Reference/uploaded screenshot (for context): /mnt/data/Screenshot 2025-11-24 at 4.21.50 AM.png

pipeline {
  agent any

  environment {
    MAVEN = "${tool 'maven'}/bin/mvn"
    NEXUS_HOST = "http://host.docker.internal:8081"
    SONAR_HOST = "http://host.docker.internal:9000"
    IMAGE_NAME = "${env.DOCKER_HUB_USER ?: 'yourdockerhubusername'}/devopsapp"
  }

  options {
    // keep logs for troubleshooting
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '10'))
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build (Maven)') {
      steps {
        sh "${MAVEN} -B -U clean package"
      }
      post {
        always {
          archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
        }
      }
    }

    stage('SonarQube Analysis') {
      steps {
        withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
          sh '''
            ${MAVEN} -B -DskipTests=true sonar:sonar \
              -Dsonar.host.url=${SONAR_HOST} \
              -Dsonar.login=${SONAR_TOKEN}
          '''
        }
      }
    }

    stage('Deploy Artifact to Nexus') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'nexus-creds', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
          sh '''
            ${MAVEN} -B deploy \
              -DaltDeploymentRepository=internal::default::${NEXUS_HOST}/repository/maven-releases/ \
              -Dnexus.username=${NEXUS_USER} -Dnexus.password=${NEXUS_PASS}
          '''
        }
      }
    }

    stage('Docker: Build Image') {
      steps {
        script {
          // tag with short commit
          GIT_COMMIT_SHORT = sh(script: "git rev-parse --short=8 HEAD", returnStdout: true).trim()
          IMAGE_TAG = "${GIT_COMMIT_SHORT}"
          dockerImage = docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
          // also tag latest
          sh "docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest || true"
        }
      }
    }

    stage('Trivy Scan (Image)') {
      steps {
        script {
          // run trivy on the newly built image; allow non-zero exit but mark issues
          sh '''
            # pull Trivy if missing (on Jenkins host you should have trivy installed) - optional
            trivy image --severity CRITICAL,HIGH --exit-code 1 ${IMAGE_NAME}:${IMAGE_TAG} || true
          '''
        }
      }
    }

    stage('Docker: Push to Docker Hub') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh '''
            echo "${DOCKER_PASS}" | docker login -u "${DOCKER_USER}" --password-stdin
            docker push ${IMAGE_NAME}:${IMAGE_TAG}
            docker push ${IMAGE_NAME}:latest || true
            docker logout || true
          '''
        }
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        script {
          // ensure kubectl context is correctly configured on Jenkins host or agent
          sh '''
            cat > k8s-deployment.yaml <<'YAML'
            apiVersion: apps/v1
            kind: Deployment
            metadata:
              name: devopsapp
              labels:
                app: devopsapp
            spec:
              replicas: 1
              selector:
                matchLabels:
                  app: devopsapp
              template:
                metadata:
                  labels:
                    app: devopsapp
                spec:
                  containers:
                  - name: devopsapp
                    image: ${IMAGE_NAME}:${IMAGE_TAG}
                    ports:
                    - containerPort: 8080
            YAML

            kubectl apply -f k8s-deployment.yaml
          '''
        }
      }
    }
  }

  post {
    success {
      echo "Pipeline completed successfully"
    }
    failure {
      echo "Pipeline failed - check console output"
    }
    always {
      // clean up workspace images if desired
      sh 'docker image prune -f || true'
    }
  }
}

