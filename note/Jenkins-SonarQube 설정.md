## AS-IS Script
```
stage("Git Clone") {
    steps {
        script {
            echo 'Git Clone'    
            git credentialsId: "${GIT_CREDENTIALS_ID}", url: "http://${GITLAB_IP}/${GITLAB_GROUP}/${GITLAB_PROJECT}.git", branch: "${BRANCH}", poll: true, changelog: true
        }
    }
}
stage("Maven Build") {
    steps {
        script {
            echo 'Build'
            sh  '''
                ${MAVEN_HOME}/bin/mvn clean package -DskipTests -P ${BRANCH} -s /var/lib/jenkins/workspace/${PROJECT_PATH}/settings.xml
                ls -al
            '''
        }
    }
}
stage('SonarQube Analysis') {
    steps {
        withCredentials([string(credentialsId: 'cicdsonarqube', variable: 'SONAR_TOKEN')]) {
            sh "${MAVEN_HOME}/bin/mvn verify -P ${BRANCH} -s /var/lib/jenkins/workspace/${PROJECT_PATH}/settings.xml sonar:sonar -Dsonar.host.url=${SONAR_URL} -Dsonar.projectName=${SONAR_PROJECT_NAME} -Dsonar.projectKey=${SONAR_PROJECT_KEY} -Dsonar.login=${SONAR_TOKEN}"
        }
    }
}
stage('SonarQube Quality Gate') {
    steps {
        script {
            // Fetch the quality gate status
            withCredentials([string(credentialsId: 'cicdsonarqube', variable: 'SONAR_TOKEN')]) {
                def qgJson = sh(script: "curl -s -u ${SONAR_TOKEN}: ${SONAR_URL}/api/qualitygates/project_status?projectKey=${SONAR_PROJECT_KEY}", returnStdout: true).trim()
                def qg = new groovy.json.JsonSlurperClassic().parseText(qgJson)

                // Extract the error reasons if available
                def errorReasons = []
                qg?.projectStatus?.conditions?.findAll { it.status == 'ERROR' }?.each {
                    def formattedErrorThreshold = it.metricKey.contains("coverage") ? "${it.errorThreshold}%" : (it.metricKey.contains("duplicated") ? "${it.errorThreshold}%" : it.errorThreshold)
                    errorReasons << "${it.metricKey}: ${it.actualValue}(${formattedErrorThreshold})"
                }

                // Print error reasons
                if (errorReasons) {
                    println "Error Reasons: ${errorReasons.join(', ')}"
                    // Set GitLab commit status to failed
                    updateGitlabCommitStatus(name: "SonarQube Quality Gate", state: "failed")
                    error "Pipeline aborted due to quality gate failure. Error Reasons: ${errorReasons.join(', ')}"
                } else {
                    println "No specific error reason available."
                }
            }
        }
    }
}
stage("Deploy") {
    steps {
        script {
            echo "Deploy"
            sshagent (credentials: ["${SSH_CREDENTIALS_ID}"]){
                sh "scp -o StrictHostKeyChecking=no -P 22222 /var/lib/jenkins/workspace/${PROJECT_PATH}/target/${WAR_FILE_NAME} ${TARGET_HOST_SSH}:/app/${WAR_FILE_NAME}"
                sh """
                    ssh -o StrictHostKeyChecking=no -p 22222 ${TARGET_HOST_SSH} '
                    ls -al /app/
                    '
                    """                                              
            }
        }
    }
}
```

