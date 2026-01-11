pipeline {
    agent any
    
    tools {
        // Note: this should match with the tool name configured in your jenkins instance (JENKINS_URL/configureTools/)
        maven "MVN_HOME"
    }
    
    environment {
        // This can be nexus3 or nexus2
        NEXUS_VERSION = "nexus3"
        // This can be http or https
        NEXUS_PROTOCOL = "http"
        // Where your Nexus is running
        NEXUS_URL           = '15.207.103.131:8081'       // no trailing slash
        NEXUS_REPOSITORY    = 'simple_customerapp'        // Make sure this repo exists in Nexus
        NEXUS_CREDENTIAL_ID = 'nexus-server'
        SCANNER_HOME = tool 'sonar-scanner'
    }
    
    stages {
        stage("clone code") {
            steps {
                script {
                    // Let's clone the source
                    git 'https://github.com/betawins/sabear_simplecutomerapp.git'
                }
            }
        }
        
        stage("mvn build") {
            steps {
                script {
                    // If you are using Windows then you should use "bat" step
                    // Since unit testing is out of the scope we skip them
                    sh 'mvn -Dmaven.test.failure.ignore=true clean install'
                }
            }
        }
        
        stage('SonarCloud') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh """
                        \$SCANNER_HOME/bin/sonar-scanner \\
                        -Dsonar.projectKey=Ncodeit \\
                        -Dsonar.projectName=Ncodeit \\
                        -Dsonar.projectVersion=2.0 \\
                        -Dsonar.sources=/var/lib/jenkins/workspace/\$JOB_NAME/src/ \\
                        -Dsonar.binaries=target/classes/com/visualpathit/account/controller/ \\
                        -Dsonar.junit.reportsPath=target/surefire-reports \\
                        -Dsonar.jacoco.reportPath=target/jacoco.exec \\
                        -Dsonar.java.binaries=src/com/room/sample
                    """
                }
            }
        }
        
        stage("publish to nexus") {
            steps {
                script {
                    // Extract POM values using Maven (reliable, no plugin)
                    def pomGroupId    = sh(script: "mvn help:evaluate -Dexpression=project.groupId -q -DforceStdout", returnStdout: true).trim()
                    def pomArtifactId = sh(script: "mvn help:evaluate -Dexpression=project.artifactId -q -DforceStdout", returnStdout: true).trim()
                    def pomVersion    = sh(script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout", returnStdout: true).trim()
                    def pomPackaging  = sh(script: "mvn help:evaluate -Dexpression=project.packaging -q -DforceStdout", returnStdout: true).trim()

                    // Find the WAR file using shell (simple glob, assumes one main artifact)
                    def artifactPath = sh(script: "ls target/*.${pomPackaging} | head -n 1", returnStdout: true).trim()

                    if (artifactPath && fileExists(artifactPath)) {
                        echo "*** File: ${artifactPath}, group: ${pomGroupId}, packaging: ${pomPackaging}, version: ${pomVersion}"

                        nexusArtifactUploader(
                            nexusVersion: NEXUS_VERSION,
                            protocol: NEXUS_PROTOCOL,
                            nexusUrl: NEXUS_URL,
                            groupId: pomGroupId,
                            version: pomVersion,
                            repository: NEXUS_REPOSITORY,
                            credentialsId: NEXUS_CREDENTIAL_ID,
                            artifacts: [
                                [artifactId: pomArtifactId, classifier: '', file: artifactPath, type: pomPackaging],
                                [artifactId: pomArtifactId, classifier: '', file: "pom.xml", type: "pom"]
                            ]
                        )
                    } else {
                        error "No artifact found in target/ with packaging ${pomPackaging}"
                    }
                }
            }
        }
    }
}