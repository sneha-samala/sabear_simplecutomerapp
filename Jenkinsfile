pipeline {
    agent any
    
    tools {
        maven 'MVN_HOME'  // Must match the exact name in Global Tool Configuration
    }

    environment {
        // Nexus Configuration
        NEXUS_VERSION       = 'nexus3'
        NEXUS_PROTOCOL      = 'http'
        NEXUS_URL           = '15.207.103.131:8081'       // no trailing slash
        NEXUS_REPOSITORY    = 'simple_customerapp'        // Make sure this repo exists in Nexus
        NEXUS_CREDENTIAL_ID = 'nexus-server'

        ARTIFACT_DIR        = 'target'
    }

    stages {
        stage('Clone Repository') {
            steps {
                git url: 'https://github.com/sneha-samala/sabear_simplecutomerapp.git',
                    branch: 'master'
            }
        }

        stage('Build with Maven') {
            steps {
                // Using -DskipTests since there are no tests in the project
                sh 'mvn clean install -DskipTests'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    // Get SonarScanner installation path (must match name in Global Tool Configuration)
                    def scannerHome = tool name: 'sonar-scanner',
                                         type: 'hudson.plugins.sonar.SonarRunnerInstallation'

                    withSonarQubeEnv('sonar') {  // 'sonar' must match your SonarQube server name in Jenkins config
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=ncodeit \
                            -Dsonar.projectName="Ncodeit Project" \
                            -Dsonar.projectVersion=${BUILD_NUMBER} \
                            -Dsonar.sources=src/main/java \
                            -Dsonar.java.binaries=target/classes
                            # Test-related parameters removed because src/test/java doesn't exist
                        """
                    }
                }
            }
        }

        stage('Publish Artifact to Nexus') {
            when {
                expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
            }
            steps {
                script {
                    def pom = readMavenPom file: 'pom.xml'
                    
                    def artifactFiles = findFiles(glob: "${ARTIFACT_DIR}/*.${pom.packaging}")
                    
                    if (artifactFiles?.size() > 0) {
                        def mainArtifact = artifactFiles[0]
                        
                        echo "Publishing artifact to Nexus:"
                        echo "→ File: ${mainArtifact.name}"
                        echo "→ GroupId: ${pom.groupId}"
                        echo "→ ArtifactId: ${pom.artifactId}"
                        echo "→ Version: ${pom.version}"
                        echo "→ Type: ${pom.packaging}"

                        nexusArtifactUploader(
                            nexusVersion: NEXUS_VERSION,
                            protocol: NEXUS_PROTOCOL,
                            nexusUrl: NEXUS_URL,
                            groupId: pom.groupId,
                            version: pom.version,
                            repository: NEXUS_REPOSITORY,
                            credentialsId: NEXUS_CREDENTIAL_ID,
                            artifacts: [
                                // Main artifact (.war in your case)
                                [
                                    artifactId: pom.artifactId,
                                    classifier: '',
                                    file: mainArtifact.path,
                                    type: pom.packaging
                                ],
                                // POM file
                                [
                                    artifactId: pom.artifactId,
                                    classifier: '',
                                    file: 'pom.xml',
                                    type: 'pom'
                                ]
                            ]
                        )
                    } else {
                        error "No main artifact found in ${ARTIFACT_DIR} with packaging ${pom.packaging}"
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()  // Clean workspace after build to save disk space
        }
        success {
            echo "Pipeline completed successfully! Artifact published to Nexus."
        }
        failure {
            echo "Pipeline failed. Please check the console output for details."
        }
    }
}
