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
                    //sh '''
                      //  docker run --rm -v $(pwd):/path -v $(pwd)/.gitleaks.toml:/.gitleaks.toml zricethezav/gitleaks:latest detect --source /path --config /.gitleaks.toml --report-format json --report-path /path/gitleaks-report.json || true
                      //  '''
                   sh '''
                        docker run --rm \
                            -v $(pwd):/path \
                            ghcr.io/gitleaks/gitleaks:latest detect \
                            --source=/path \
                            --report-format=json \
                            --report-path=/path/gitleaks-report.json \
                            --verbose \
                            --no-git \
                            --max-target-megabytes=0 \
                            --additional-locations=/path
                    '''
                    
                    // Archive the reports as artifacts
                        archiveArtifacts artifacts: 'gitleaks-report.json', allowEmptyArchive: true


                }
                 // Display the contents of the report in a separate step
                //script {
                  //  echo "Gitleaks Report:"
                  //  sh 'cat gitleaks-report.json || echo "Report not found or empty."'
                //}
            }
  }
  }
}
