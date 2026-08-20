pipeline {
    agent any

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    tools {
        maven 'maven3'
        jdk 'jdk-17'
    }

    stages {

        stage('git checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/rijunandi-repo/Ekart.git'
            }
        }

        stage('compile') {
            steps {
                sh 'mvn compile'
            }
        }

        stage('unit tests') {
            steps {
                sh 'mvn test -DskipTests=true'
            }
        }

        stage('SonarQube analysis') {
            steps {
                withSonarQubeEnv('sonar-scanner') {
                    sh '''
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=EKART \
                        -Dsonar.projectName=EKART \
                        -Dsonar.java.binaries=target/classes
                    '''
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'nvd-api-key',
                        variable: 'NVD_API_KEY'
                    )
                ]) {
                    dependencyCheck(
                        additionalArguments: "--nvdApiKey=$NVD_API_KEY",
                        odcInstallation: 'DC'
                    )
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn package -DskipTests=true'
            }
        }

        stage('deploy to Nexus') {
            steps {
                withMaven(
                    globalMavenSettingsConfig: 'global-maven',
                    jdk: 'jdk-17',
                    maven: 'maven3',
                    mavenSettingsConfig: '',
                    traceability: true
                ) {
                    sh 'mvn deploy -DskipTests=true'
                }
            }
        }

        stage('build and Tag docker image') {
            steps {
                sh 'docker build -t rijunandi/ekart:latest -f docker/Dockerfile .'
            }
        }

        stage('Push image to Hub') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'dockerhub-pwd',
                        variable: 'dockerhubpwd'
                    )
                ]) {
                    sh '''
                        echo "$dockerhubpwd" | docker login \
                        -u rijunandi \
                        --password-stdin

                        docker push rijunandi/ekart:latest
                    '''
                }
            }
        }

        stage('EKS and Kubectl configuration') {
            steps {
                sh '''
                    mkdir -p /var/lib/jenkins/.kube

                    aws eks update-kubeconfig \
                      --region ap-south-1 \
                      --name project-cluster \
                      --kubeconfig /var/lib/jenkins/.kube/config

                    kubectl \
                      --kubeconfig /var/lib/jenkins/.kube/config \
                      get nodes
                '''
            }
        }

        stage('Deploy to k8s') {
            steps {
                sh '''
                    kubectl \
                      --kubeconfig /var/lib/jenkins/.kube/config \
                      apply -f deploymentservice.yml
                '''
            }
        }
    }
}
