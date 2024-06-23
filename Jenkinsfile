pipeline {
    agent {label "Dev-Server}"

    environment {
        SONAR_HOME= tool "Sonar"
        DOCKER_CREDENTIALS_ID = 'dockerhub-credentials'
        KUBECONFIG_CREDENTIALS_ID = 'kubeconfig'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage("SonarQube Quality Analysis"){
            steps{
                withSonarQubeEnv("Sonar"){
                    sh "$SONAR_HOME/bin/sonar-scanner -Dsonar.projectName=nodejs -Dsonar.projectKey=nodejs"
                }
            }
        }

        stage("OWASP Dependency Check"){
            steps{
                dependencyCheck additionalArguments: '--scan ./', odcInstallation: 'dc'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage("Sonar Quality Gate Scan"){
            steps{
                timeout(time: 2, unit: "MINUTES"){
                    waitForQualityGate abortPipeline: false
                }
            }
        }
        stage("Trivy File System Scan"){
            steps{
                sh "trivy fs --format  table -o trivy-fs-report.html ."
            }
        }

        stage('Build and Test') {
            steps {
                    sh "docker build -t node.js-app ."
                    echo "code built and tested successfully"
            }
        }

       stage("Push"){
            steps {
                echo "Pushing the image to docker hub"
                withCredentials([usernamePassword(credentialsId:"dockerHub",passwordVariable:"dockerHubPass",usernameVariable:"dockerHubUser")]){
                sh "docker tag node.js-app ${env.dockerHubUser}/node.js-app:latest"
                sh "docker login -u ${env.dockerHubUser} -p ${env.dockerHubPass}"
                sh "docker push ${env.dockerHubUser}/node.js-app:latest"
                }
            }
        }

       stage('Deploy') {
    steps {
        withCredentials([file(credentialsId: KUBECONFIG_CREDENTIALS_ID, variable: 'KUBECONFIG')]) {
            sh 'helm upgrade --install node.js-app ./helm-chart --set image.repository=${env.dockerHubUser}/node.js-app --set image.tag=latest --kubeconfig $KUBECONFIG'
        }
    }
}


    post {
        always {
            cleanWs()
        }
    }
}
