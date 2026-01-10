pipeline {
    agent any
    
    tools {
        maven 'MVN_HOME'
    }

    environment {
        NEXUS_VERSION       = 'nexus3'
        NEXUS_PROTOCOL      = 'http'
        NEXUS_URL           = '15.207.103.131:8081'
        NEXUS_REPOSITORY    = 'simple_customerapp'
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
                sh 'mvn clean install -DskipTests'
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
                        
                        echo "Publishing to Nexus:"
                        echo "→ File:      ${mainArtifact.name}"
                        echo "→ Group:     ${pom.groupId}"
                        echo "→ Artifact:  ${pom.artifactId}"
                        echo "→ Version:   ${pom.version}"
                        echo "→ Packaging: ${pom.packaging}"

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
                        error "No artifact found: ${ARTIFACT_DIR}/*.${pom.packaging}"
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
            echo "Pipeline completed successfully! WAR file published to Nexus."
        }
        failure {
            echo "Pipeline failed. Check console logs."
        }
    }
}
