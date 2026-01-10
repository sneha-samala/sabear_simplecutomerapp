pipeline {
    agent any
    
    tools {
        maven 'MVN_HOME'  // Only Maven here – that's fine
    }

    environment {
        // Nexus Configuration
        NEXUS_VERSION       = 'nexus3'
        NEXUS_PROTOCOL      = 'http'
        NEXUS_URL           = '15.207.103.131:8081'       // no trailing slash
        NEXUS_REPOSITORY    = 'simple_customerapp'        // confirm exists & is maven-hosted
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

        stage('Build & Test with Maven') {
            steps {
                sh 'mvn clean verify'  // runs tests + generates JaCoCo report for Sonar
                // Use 'mvn clean install -DskipTests' only if you really need to skip tests
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    // Fetch the SonarScanner installation path (this is the correct way)
                    def scannerHome = tool name: 'sonar-scanner',   // ← MUST match EXACT name in Global Tool Config > SonarQube Scanner > Name
                                     type: 'hudson.plugins.sonar.SonarRunnerInstallation'

                    withSonarQubeEnv('sonar') {  // 'sonar' = your SonarQube server name in Configure System > SonarQube servers
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=ncodeit \
                            -Dsonar.projectName="Ncodeit Project" \
                            -Dsonar.projectVersion=${BUILD_NUMBER} \
                            -Dsonar.sources=src/main/java \
                            -Dsonar.java.binaries=target/classes \
                            -Dsonar.tests=src/test/java \
                            -Dsonar.junit.reportsPath=target/surefire-reports \
                            -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
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
                        
                        echo "Uploading → ${mainArtifact.name} (v${pom.version})"
                        
                        nexusArtifactUploader(
                            nexusVersion: NEXUS_VERSION,
                            protocol: NEXUS_PROTOCOL,
                            nexusUrl: NEXUS_URL,
                            groupId: pom.groupId,
                            version: pom.version,
                            repository: NEXUS_REPOSITORY,
                            credentialsId: NEXUS_CREDENTIAL_ID,
                            artifacts: [
                                [artifactId: pom.artifactId, classifier: '', file: mainArtifact.path, type: pom.packaging],
                                [artifactId: pom.artifactId, classifier: '', file: 'pom.xml', type: 'pom']
                            ]
                        )
                    } else {
                        error "Artifact not found: ${ARTIFACT_DIR}/*.${pom.packaging}"
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo "Pipeline SUCCESS ✓ Artifact published!"
        }
        failure {
            echo "Pipeline FAILED ✗ Check logs."
        }
    }
}
