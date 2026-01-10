pipeline {
    agent any
    
    tools {
        maven 'MVN_HOME'                    // Must match name in Global Tool Configuration
        sonarscanner 'sonar-scanner'        // ← IMPORTANT: Add this! Must match the Name you set in SonarQube Scanner installations
    }

    environment {
        // Nexus Configuration
        NEXUS_VERSION       = 'nexus3'
        NEXUS_PROTOCOL      = 'http'
        NEXUS_URL           = '15.207.103.131:8081'       // no trailing slash
        NEXUS_REPOSITORY    = 'simple_customerapp'        // confirm this repo exists in Nexus
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
                // Changed to 'verify' to run tests + generate JaCoCo report
                sh 'mvn clean verify'
                // If you really want to skip tests → use: mvn clean install -DskipTests
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    // Option 1: Recommended - Jenkins auto-resolves the scanner path
                    withSonarQubeEnv('sonar') {   // 'sonar' = name of your SonarQube server config
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

                    // Alternative (more explicit - use if above fails):
                    // def scannerHome = tool 'sonar-scanner'
                    // withSonarQubeEnv('sonar') {
                    //     sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=..."
                    // }
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
                        
                        echo "Uploading artifact → ${mainArtifact.name}"
                        echo "GroupId    : ${pom.groupId}"
                        echo "ArtifactId : ${pom.artifactId}"
                        echo "Version    : ${pom.version}"
                        echo "Packaging  : ${pom.packaging}"

                        nexusArtifactUploader(
                            nexusVersion: NEXUS_VERSION,
                            protocol: NEXUS_PROTOCOL,
                            nexusUrl: NEXUS_URL,
                            groupId: pom.groupId,
                            version: pom.version,
                            repository: NEXUS_REPOSITORY,
                            credentialsId: NEXUS_CREDENTIAL_ID,
                            artifacts: [
                                [
                                    artifactId: pom.artifactId,
                                    classifier: '',
                                    file: mainArtifact.path,
                                    type: pom.packaging
                                ],
                                [
                                    artifactId: pom.artifactId,
                                    classifier: '',
                                    file: 'pom.xml',
                                    type: 'pom'
                                ]
                            ]
                        )
                    } else {
                        error "Main artifact not found → ${ARTIFACT_DIR}/*.${pom.packaging}"
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()  // clean workspace to save disk space
        }
        success {
            echo "Pipeline SUCCESS ✓ Artifact published to Nexus!"
        }
        failure {
            echo "Pipeline FAILED ✗ Check console output for details."
        }
    }
}
