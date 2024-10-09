pipeline {
    agent {
        label 'jenkins-slave'  // Replace with the label of your slave node
    }
 environment{
        DOCKER_REGISTRY = 'https://index.docker.io/v1/'
        DOCKER_IMAGE = 'aatikah/task-app'
        remoteHost = '34.133.55.7'
	remoteHostInternal = '10.0.1.14'
        //DEFECTDOJO_API_KEY = credentials('DEFECTDOJO_API_KEY')
        //DEFECTDOJO_URL = 'http://34.42.127.145:8080'
       // PRODUCT_NAME = 'django-project'
    }
    
  stages{
      
    stage('Testing Node') {
            steps {
                script {  
                    sh 'echo "Hello from Node"' 
               }
            }
            }
      
       stage('Run Gitleaks with Custom Config') {
            steps {
                script {
                    // Pull and run the Gitleaks Docker image with a custom config file
                    sh '''
                        docker run --rm -v $(pwd):/path -v $(pwd)/.gitleaks.toml:/.gitleaks.toml zricethezav/gitleaks:latest detect --source /path --config /.gitleaks.toml --report-format json --report-path /path/gitleaks-report.json || true
                        '''
                    // Archive the reports as artifacts
                        archiveArtifacts artifacts: 'gitleaks-report.json', allowEmptyArchive: true
                }
                 // Display the contents of the report in a separate step
                script {
                    echo "Gitleaks Report:"
                    sh 'cat gitleaks-report.json || echo "Report not found or empty."'
                }
            }
  }
      
      stage('Source Composition Analysis'){
            steps{
                script{
                    sh 'rm owasp* || true'
                    sh 'wget "https://raw.githubusercontent.com/aatikah/sumayyah/refs/heads/master/owasp-dependency-check.sh"'
                    sh 'bash owasp-dependency-check.sh'
                    // Archive the reports as artifacts
                        archiveArtifacts artifacts: 'dependency-check-report.json,dependency-check-report.html,dependency-check-report.xml', allowEmptyArchive: true

            // Publish HTML report
                    publishHTML(target: [
                        allowMissing: false,
                        alwaysLinkToLastBuild: false,
                        keepAll: true,
                        reportDir: '.',
                        reportFiles: 'dependency-check-report.html',
                        reportName: 'OWASP Dependency Checker Report'
                    ])

                    // Parse JSON report to check for issues
           // This if block can be added in another script block outside this script block to fail pipeline if cvssv is above 7
                if (fileExists('dependency-check-report.json')) {
                    def jsonReport = readJSON file: 'dependency-check-report.json'
                    def vulnerabilities = jsonReport.dependencies.collect { it.vulnerabilities ?: [] }.flatten()
                    def highVulnerabilities = vulnerabilities.findAll { it.cvssv3?.baseScore >= 7 }
                    echo "OWASP Dependency-Check found ${vulnerabilities.size()} vulnerabilities, ${highVulnerabilities.size()} of which are high severity (CVSS >= 7.0)"
                } else {
                    echo "Dependency-Check JSON report not found. The scan may have failed."
                }
                }
            }
        }

      //BANDIT STAGE

    
stage('Build and Push Docker Image') {

            steps {
                script {
                    // Wrap the Docker commands with withCredentials to securely access the Docker credentials
                    withCredentials([usernamePassword(credentialsId: 'DOCKER_LOGIN', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        // Build the Docker image
                        sh "docker build -t ${DOCKER_IMAGE} ."


                        // Log in to the Docker registry using a more secure method. set +x set -x This turns off command echoing temporarily
                        sh '''
                            set +x
                            echo "$DOCKER_PASSWORD" | docker login $DOCKER_REGISTRY -u "$DOCKER_USERNAME" --password-stdin
                            set -x
                        '''
                       
                        // Push the Docker image
                        sh "docker push ${DOCKER_IMAGE}"
                        
                        // Log out from the Docker registry
                        sh "docker logout $DOCKER_REGISTRY"

                        // Clean up: remove any leftover Docker credentials
                        sh "rm -f /home/jenkins/.docker/config.json"
                    }
                }
            }
        }
      
    

      stage('Deploy to GCP VM') {
    steps {
        script {
            def remoteUser = 'jenkins-slave'
	    def dockerImage = 'aatikah/task-app'
            
            sshagent(['JENKINS_MASTER_KEY_2']) {
                // Stop and remove the old container if it existS
                sh """
                    ssh -o StrictHostKeyChecking=no ${remoteUser}@${remoteHostInternal} '
                        container_id=\$(docker ps -q --filter ancestor=${dockerImage})
                        if [ ! -z "\$container_id" ]; then
                            docker stop \$container_id
                            docker rm \$container_id
                        fi
                    '
                """
                
                // Pull the latest image and run the new container
                sh """
                    ssh -o StrictHostKeyChecking=no ${remoteUser}@${remoteHostInternal} '
                        docker pull ${dockerImage} && 
                        docker run -d --restart unless-stopped -p 8000:8000 --name my-task-app ${dockerImage}
                    '
                """
                
                // Verify the deployment
                sh """
                    ssh -o StrictHostKeyChecking=no ${remoteUser}@${remoteHostInternal} '
                        if docker ps | grep -q ${dockerImage}; then
                            echo "Deployment successful"
                        else
                            echo "Deployment failed"
                            exit 1
                        fi
                    '
                """
            }
        }
    }
}
      stage('DAST OWASP ZAP Scan') {
    steps {
        script {
            def zapHome ='/opt/zaproxy' // Path to ZAP installation
            def reportNameHtml = "zap-scan-report.html"
            def reportNameXml = "zap-scan-report.xml"
	    def reportNameJson = "zap-scan-report.json"
            
            // Perform ZAP scan
            sh """
            ${zapHome}/zap.sh -cmd \
                -quickurl http://${remoteHost} \
                -quickprogress \
                -quickout ${WORKSPACE}/${reportNameHtml} 

             ${zapHome}/zap.sh -cmd \
                -quickurl http://${remoteHost} \
                -quickprogress \
                -quickout ${WORKSPACE}/${reportNameXml}

  	     ${zapHome}/zap.sh -cmd \
		-quickurl http://${remoteHost} \
		-quickprogress \
		-quickout ${WORKSPACE}/${reportNameJson}
                """
          
            // Archive the ZAP reports
            archiveArtifacts artifacts: "${reportNameHtml},${reportNameXml}, ${reportNameJson}", fingerprint: true
          
            // Read and parse the HTML report
            def htmlReportContent = readFile(reportNameHtml)

            // Example: Check for high alerts in HTML using regex (customize based on your report structure)
            def highRiskPattern = ~/<span class="risk"><strong>High<\/strong><\/span>.*?<a href="(.*?)">(.*?)<\/a>/
            def highAlerts = []

            htmlReportContent.eachMatch(highRiskPattern) { match ->
                highAlerts.add([url: match[1], alert: match[2]])
            }
            
            if (highAlerts.size() > 0) {
                echo "Found ${highAlerts.size()} high-risk vulnerabilities!"
                highAlerts.each { alert ->
                    echo "High Risk Alert: ${alert.alert} at ${alert.url}"
                }
                // Exit with code 1 if high-risk vulnerabilities are found
                error "OWASP ZAP scan found high-risk vulnerabilities. Check the ZAP report for details."
            }else {
                echo "No high-risk vulnerabilities found."
            }
            
            // Read and parse JSON report
            def zapJson = readJSON file: reportNameJson
            
            // Example: Check for high alerts in JSON
            def highAlerts = zapJson.site[0].alerts.findAll { it.riskcode >= 3 }
            
            if (highAlerts.size() > 0) {
                echo "Found ${highAlerts.size()} high-risk vulnerabilities!"
                highAlerts.each { alert ->
                    echo "High Risk Alert: ${alert.alert} at ${alert.url}"
                }
                error "OWASP ZAP scan found high-risk vulnerabilities. Check the ZAP report for details."
            }

             // Publish HTML report
            publishHTML(target: [
                allowMissing: false,
                alwaysLinkToLastBuild: false,
                keepAll: true,
                reportDir: '.',
                reportFiles: 'zap-scan-report.html',
                reportName: "ZAP Security Report"
            ])
            
            // Optional: Publish JSON report as a build artifact
            //archiveArtifacts artifacts: 'zap-scan-report.json', fingerprint: true

            
        }
    }
   
           
     
}
  }
}
