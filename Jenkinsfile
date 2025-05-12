def COLOR_MAP = [
    'SUCCESS': 'good', 
    'FAILURE': 'danger',
]
pipeline {
    agent any

    tools {
        maven "Maven"
        jdk "JDK17"
    }
    environment {
        NEXUS_VERSION = "nexus3"
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "localhost:8081"
        NEXUS_REPOSITORY = "vprofile-repo"
        NEXUS_REPO_ID = "vprofile-repo"
        NEXUS_CREDENTIAL_ID = "nexus-id"
        ARTVERSION = "${env.BUILD_ID}"
        IMAGE_NAME = "bassma/vprofile-app"
        IMAGE_TAG  = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'master', url: 'https://github.com/BassmaMandour/Vprofile-Jenkins.git'
            }
        }

        stage('BUILD') {
            steps {
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    echo 'Now Archiving...'
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }

        stage('UNIT TEST') {
            steps {
                sh 'mvn test'
            }
        }

        stage('INTEGRATION TEST') {
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }

        stage('CODE ANALYSIS WITH CHECKSTYLE') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }
        

        stage('CODE ANALYSIS with SONARQUBE') {
            environment {
                scannerHome = tool 'sonar-pro'
            }
            steps {
                withSonarQubeEnv('sonar-server') {
                    withCredentials([string(credentialsId: 'sonar-id', variable: 'SONAR_TOKEN')]) {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                                -Dsonar.projectKey=vprofile \
                                -Dsonar.projectName=vprofile-repo \
                                -Dsonar.projectVersion=1.0 \
                                -Dsonar.sources=src/ \
                                -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                                -Dsonar.junit.reportsPath=target/surefire-reports/ \
                                -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                                -Dsonar.token=$SONAR_TOKEN \
                                -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml \
                                -Dsonar.host.url=http://localhost:9000
                        """
                    }
                }
            }
        }
        stage("Publish to Nexus Repository Manager") {
            steps {
                    script {
                        def pom = readMavenPom file: "pom.xml"
                        def filesByGlob = findFiles(glob: "target/*.${pom.packaging}")
                        def artifactPath = filesByGlob[0]?.path
                        def artifactExists = fileExists(artifactPath)

                        if (artifactExists) {
                            echo "*** File: ${artifactPath}, group: ${pom.groupId}, packaging: ${pom.packaging}, version: ${pom.version}"
                            nexusArtifactUploader(
                                nexusVersion: NEXUS_VERSION,
                                protocol: NEXUS_PROTOCOL,
                                nexusUrl: NEXUS_URL,
                                groupId: pom.groupId,
                                version: ARTVERSION,
                                repository: NEXUS_REPOSITORY,
                                credentialsId: NEXUS_CREDENTIAL_ID,
                                artifacts: [
                                    [artifactId: pom.artifactId,
                                     classifier: '',
                                     file: artifactPath,
                                     type: pom.packaging],
                                    [artifactId: pom.artifactId,
                                     classifier: '',
                                     file: "pom.xml",
                                     type: "pom"]
                                ]
                            )
                        } else {
                            error "*** File: ${artifactPath}, could not be found"
                        }
                    }
                }
            }
        stage('Build App Image') {
          steps {
            script {
                dockerImage = docker.build(
                  "${IMAGE_NAME}:${env.BUILD_NUMBER}",
                  "-f Docker/dockerfile ."
                )
            }
          }
        }
        
        stage('Push to Docker Hub') {
          steps {
            script {
              docker.withRegistry('https://registry.hub.docker.com', 'docker-hub') {
                docker.image("${IMAGE_NAME}:${IMAGE_TAG}").push()
              }
            }
          }
        }

      
    }
    post {
        always {
            echo 'Slack Notifications.'
            slackSend channel: '#all-devops-cicd',
                color: COLOR_MAP[currentBuild.currentResult],
                message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
        }
    }
}
