pipeline {
    agent any
    tools {
        maven 'maven'
    }
    environment {
        SCANNER_HOME = tool 'sonarqube'
        TRIVY_TIMEOUT = '30m'
    }
    stages {
        stage('GIT SCM') {
            steps {
                git branch: 'dev/chow', credentialsId: 'github', url: 'https://github.com/Rajendra0609/vulnerable_application.git'
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                script {
                    // Run OWASP Dependency Check
                    dependencyCheck additionalArguments: '--scan ./ --format HTML', odcInstallation: 'dpcheck'
                    
                    // Publish the Dependency Check report
                    dependencyCheckPublisher pattern: '**/dependency-check-report.html'
                    
                    // Archive the report file for access after the build
                    archiveArtifacts artifacts: '**/dependency-check-report.html'
                }
            }
        }
        stage('Lynis Security Scan') {
            steps {
                script {
                    // Execute the Lynis security scan and convert the output to HTML
                    sh 'lynis audit system | ansi2html > lynis-report.html'
                    
                    // Display the absolute path of the report in the Jenkins console output
                    def reportPath = "${env.WORKSPACE}/lynis-report.html"
                    echo "Lynis report path: ${reportPath}"
                    
                    // Archive the report file for access after the build
                    archiveArtifacts artifacts: 'lynis-report.html'
                }
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=vulnerable \
                    -Dsonar.java.binaries=. \
                    -Dsonar.projectKey=vulnerable'''
                }
            }
        }
        stage('Build & Push Docker Image') {
            environment {
                DOCKER_IMAGE = "daggu1997/vulnerable_application:${BUILD_NUMBER}"
                REGISTRY_CREDENTIALS = credentials('docker')
            }
            steps {
                script {
                    sh 'docker build -t ${DOCKER_IMAGE} .'
                    
                    // Scan the Docker image with Trivy
                    sh 'trivy image --format table --timeout ${TRIVY_TIMEOUT} -o trivy-image-report.html ${DOCKER_IMAGE}'
                    
                    // Push the Docker image to the registry
                    docker.withRegistry('https://index.docker.io/v1/', 'docker') {
                        def dockerImage = docker.image("${DOCKER_IMAGE}")
                        dockerImage.push()
                    }
                    
                    // Remove the Docker image from local to save space
                    sh 'docker rmi ${DOCKER_IMAGE}'
                    
                    // Archive the Trivy report
                    def reportPath = "${env.WORKSPACE}/trivy-image-report.html"
                    echo "Trivy report path: ${reportPath}"
                    archiveArtifacts artifacts: 'trivy-image-report.html'
                }
            }
        }
        stage('Updating Deployment File and Creating Merge Request') {
            environment {
                GIT_REPO_NAME = "vulnerable_application"
                GIT_USER_NAME = "Rajendra0609"
                TARGET_BRANCH = "main" // or the branch you want to merge into
                SOURCE_BRANCH = "dev/chow" // or the branch you're pushing changes to
            }
            steps {
                withCredentials([string(credentialsId: 'GITHUB_TOKEN', variable: 'GITHUB_TOKEN')]) {
                    script {
                        def releaseTag = "v0.${BUILD_NUMBER}.0"
                        env.RELEASE_TAG = releaseTag // Export the variable to the shell

                        sh '''
                            git config user.email "rajendra.daggubati@gmail.com"
                            git config user.name "Rajendra0609"
                            BUILD_NUMBER=${BUILD_NUMBER}
                            git tag -a ${RELEASE_TAG} -m "Release ${RELEASE_TAG}"
                            git push --force https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} ${RELEASE_TAG}
                        '''
                    }
                }
                
                // Create a merge request
                httpRequest url: "https://api.github.com/repos/${GIT_USER_NAME}/${GIT_REPO_NAME}/pulls",
                    authentication: 'Bearer ' + GITHUB_TOKEN,
                    httpMode: 'POST',
                    contentType: 'APPLICATION_JSON',
                    requestBody: """{
                        "title": "Update deployment Image to version ${BUILD_NUMBER}",
                        "head": "${SOURCE_BRANCH}",
                        "base": "${TARGET_BRANCH}",
                        "body": "Automated merge request from Jenkins pipeline"
                    }"""
            }
        }
        stage('Cleanup Workspace') {
            steps {
                cleanWs()
                sh 'echo "Cleaned Up Workspace For Project"'
            }
        }
    }
}
