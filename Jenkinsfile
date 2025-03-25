pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = tool 'sonar'
    }

    stages {
       
        stage('git checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Chandanibps/Boardgame.git'
            }
        }

        stage('compile') {
            steps {
                sh 'mvn compile'
            }
        }

        stage('test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('trivy fs scan') {
            steps {
                sh 'trivy fs --format table -o trivy-fs-report.html .'
            }
        }

        stage('SonarQube') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh """
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=Board-game \
                        -Dsonar.projectKey=Board-game
                    """
                }
            }
        }

        stage('sonarqube qualitycheck') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }

        stage('build') {
            steps {
                sh 'mvn package'
            }
        }

        stage('publish artifacts to nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-maven', jdk: 'jdk17', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                    sh """
                        mvn clean deploy \
                            -DaltReleaseDeploymentRepository=redeploy-repo::default::http://34.130.117.231:8081/repository/maven-redeploy/
                    """
                }
            }
        }

        stage('Build and tag') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub', toolName: 'docker') {
                        sh 'docker build -t chandanibps/boardgame:latest .'
                    }
                }
            }
        }

        stage('Trivy image Scan') {
            steps {
                sh "trivy image --format table -o trivy-fs-report.html chandanibps/boardgame:latest"
            }
        }

        stage('docker push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub', toolName: 'docker') {
                        sh "docker push chandanibps/boardgame:latest"
                    }
                }
            }
        }

        stage('deploy k8') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'k8-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://10.188.0.7:6443') {
                    sh "kubectl apply -f deployment.yml"
                    sh "kubectl get svc -n webapps"
                    sh "kubectl get pods -n webapps"
                }
            }
        }
    }
    post {
        success {
            mail(
                to: 'chandanchoudhary7315@gmail.com',
                subject: "SUCCESS: ${currentBuild.fullDisplayName}",
                body: "Build details: ${env.BUILD_URL}"
            )
        }
    }
}
