pipeline {
    agent {
        label 'jenkins-slave'  // Replace with the label of your slave node
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

    stage('SAST With Bandit Security Scan') {
    steps {
        script {
           
            // Run Bandit scan and generate reports
            sh '''
                python3 -m venv bandit_venv
                . bandit_venv/bin/activate
                pip install --upgrade pip
                pip install bandit
                
            
                bandit -r . -f json -o bandit-report.json --exit-zero
                bandit -r . -f html -o bandit-report.html --exit-zero

                deactivate
            '''
            // Archive the reports as artifacts
            archiveArtifacts artifacts: 'bandit-report.json,bandit-report.html', allowEmptyArchive: true

            // Publish HTML report
            publishHTML(target: [
                allowMissing: false,
                alwaysLinkToLastBuild: false,
                keepAll: true,
                reportDir: '.',
                reportFiles: 'bandit-report.html',
                reportName: 'Bandit Security Scan Report'
            ])
            
            // Parse JSON report to check for issues
            script {
                def jsonReport = readJSON file: 'bandit-report.json'
                def issueCount = jsonReport.results.size()
                if (issueCount > 0) {
                    echo "Bandit found ${issueCount} potential security issue(s). Please review the report."
                } else {
                    echo "Bandit scan completed successfully with no issues found."
                }
            }
        }
    }
}
  }
}
