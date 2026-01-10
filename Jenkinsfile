pipeline {
    agent any
    
    tools {
        maven 'MVN_HOME'           // Must match the exact name configured in Global Tool Configuration
        // sonar_scanner is used via withSonarQubeEnv, no need to declare as tool here
    }

    environment {
        // Nexus Configuration
        NEXUS_VERSION     = 'nexus3'
        NEXUS_PROTOCOL    = 'http'
        NEXUS_URL         = '15.207.103.131:8081'          // no trailing slash
        NEXUS_REPOSITORY  = 'simple_customerapp'                    // ← is this really the target repo name?
        NEXUS_CREDENTIAL_ID = 'nexus-server'

        // You can also define these if you want to make them more visible
        ARTIFACT_DIR      = 'target'
    }

    stages {

        stage('Clone Repository') {
            steps {
                git url: 'https://github.com/sneha-samala/sabear_simplecutomerapp.git',
                    branch: 'master'   // ← add branch if not default
            }
        }

        stage('Build with Maven') {
            steps {
                sh 'mvn clean install -DskipTests'   // skipping tests is cleaner than ignore failure
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {   // ← must match Jenkins SonarQube installation name
                    sh '''
                        sonar-scanner \
                        -Dsonar.projectKey=ncodeit \
                        -Dsonar.projectName="Ncodeit Project" \
                        -Dsonar.projectVersion=${BUILD_NUMBER} \
                        -Dsonar.sources=src/main/java \
                        -Dsonar.java.binaries=target/classes \
                        -Dsonar.tests=src/test/java \
                        -Dsonar.junit.reportsPath=target/surefire-reports \
                        -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
                    '''
                }
            }
        }

        stage('Publish Artifact to Nexus') {
            when {
                expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
            }
            steps {
                script {
                    // Read POM once
                    def pom = readMavenPom file: 'pom.xml'
                    
                    // Find the main artifact (jar/war/etc)
                    def artifactFiles = findFiles(glob: "${ARTIFACT_DIR}/*.${pom.packaging}")
                    
                    if (artifactFiles?.size() > 0) {
                        def mainArtifact = artifactFiles[0]
                        
                        echo "Uploading artifact: ${mainArtifact.name}"
                        echo "→ Group: ${pom.groupId}"
                        echo "→ Artifact: ${pom.artifactId}"
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
                                // Main artifact (.jar / .war / etc)
                                [
                                    artifactId: pom.artifactId,
                                    classifier: '',
                                    file: mainArtifact.path,
                                    type: pom.packaging
                                ],
                                // POM file (very useful for transitive dependencies)
                                [
                                    artifactId: pom.artifactId,
                                    classifier: '',
                                    file: 'pom.xml',
                                    type: 'pom'
                                ]
                            ]
                        )
                    } else {
                        error "No artifact found in ${ARTIFACT_DIR} with packaging ${pom.packaging}"
                    }
                }
            }
        }
    }

    post {
        always {
            // Optional: clean workspace to save disk space
            cleanWs()
        }
        success {
            echo "Pipeline completed successfully! Artifact published to Nexus."
        }
        failure {
            echo "Pipeline failed. Check logs for details."
        }
    }
}
