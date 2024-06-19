pipeline {
    agent any
    tools {
        jdk 'jdk17'
        maven 'maven3'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/Pramod858/Mission.git'
            }
        }
        
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        
        stage('Test') {
            steps {
                sh "mvn test -DskipTests=true"
            }
        }
        
        stage('File System Scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }
        
        stage('SonarQube Analsyis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=Mission -Dsonar.projectKey=Mission \
                    -Dsonar.java.binaries=. '''
                }
            }
        }
        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }
        stage('Build') {
            steps {
                sh "mvn package -DskipTests=true"
            }
        }
        stage('Publish Artifact To Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk17', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                    sh "mvn deploy -DskipTests=true"
                }
            }
        }
        
        stage('Build & Tag Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                    sh "docker build -t pramod858/mission:v${BUILD_NUMBER} ."
                    }
                }
            }
        }
        
        stage('Docker Image Scan') {
            steps {
                sh "trivy image --format table -o trivy-image-report.html pramod858/mission:v${BUILD_NUMBER} "
            }
        }
        
        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                    sh "docker push pramod858/mission:v${BUILD_NUMBER}"
                    }
                }
            }
        }
        
        stage('Update GIT') {
            steps{
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    withCredentials([usernamePassword(credentialsId: 'git-cred', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                        //def encodedPassword = URLEncoder.encode("$GIT_PASSWORD",'UTF-8')
                        sh "git config user.email pramodbadiger45@gmail.com"
                        sh "git config user.name Pramod858"
                        //sh "git switch master"
                        sh "cat K8s-manifest/deployment-service.yaml"
                        sh "sed -i 's+pramod858/mission.*+pramod858/mission:v${BUILD_NUMBER}+g' K8s-manifest/deployment-service.yaml"
                        sh "cat K8s-manifest/deployment-service.yaml"
                        sh "git add K8s-manifest/deployment-service.yaml"
                        sh "git commit -m 'Done by Jenkins Job changemanifest:v${env.BUILD_NUMBER}'"
                        sh "git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/${GIT_USERNAME}/Mission.git HEAD:main"
                    }
                }
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                withKubeCredentials(kubectlCredentials: [[caCertificate: '', clusterName: 'my-eks', contextName: '', credentialsId: 'K8S', namespace: 'webapps', serverUrl: 'https://D50C74EB4949DA5F1B14D2849956F74A.gr7.us-east-1.eks.amazonaws.com']]) {
                    sh "kubectl apply -f K8s-manifest/deployment-service.yaml -n webapps"
                    sleep 60
                }
            }
        }
        stage('Get Kubernetes Pods Status') {
            steps {
                withKubeCredentials(kubectlCredentials: [[caCertificate: '', clusterName: 'my-eks', contextName: '', credentialsId: 'K8S', namespace: 'webapps', serverUrl: 'https://D50C74EB4949DA5F1B14D2849956F74A.gr7.us-east-1.eks.amazonaws.com']]) {
                    sh "kubectl get pods -n webapps"
                }
            }
        }

    }
     post {
        success {
            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                
                def successfulHTML = """
                    <!DOCTYPE html>
                    <html>
                    <head>
                        <title>Pipeline Successful</title>
                    </head>
                    <body style="font-family: Arial, sans-serif;">
                        <div style="background-color: #DFF0D8; padding: 20px;">
                            <h2 style="color: #3C763D;">Pipeline Successful</h2>
                            <p>Your pipeline for ${jobName} build ${buildNumber} has succeeded. Great job!</p>
                            <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                        </div>
                    </body>
                    </html>
                """
                emailext subject: "Pipeline Successful",
                            body: successfulHTML,
                            from: "jenkins@example.com",
                            to: "pramod.c.b.2001@gmail.com",
                            replyTo: "jenkins@example.com",
                            mimeType: 'text/html'
            }
        }
        failure {
            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                
                def failedHTML = """
                    <!DOCTYPE html>
                    <html>
                    <head>
                        <title>Pipeline Failed</title>
                    </head>
                    <body style="font-family: Arial, sans-serif;">
                        <div style="background-color: #F2DEDE; padding: 20px;">
                            <h2 style="color: #A94442;">Pipeline Failed</h2>
                            <p>Your pipeline for ${jobName} build #${buildNumber} has failed. Please check and take necessary actions.</p>
                            <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                        </div>
                    </body>
                    </html>
                """
                emailext subject: "Pipeline Failed",
                            body: failedHTML,
                            from: "jenkins@example.com",
                            to: "pramod.c.b.2001@gmail.com",
                            replyTo: "jenkins@example.com",
                            mimeType: 'text/html'
            }
        }
    }
    
}