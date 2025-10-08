pipeline {
    agent none
    stages {
        stage('Checkout Source Code') {
            agent any
            steps {
                // Manually define the Git checkout instead of using checkout scm
                git branch: 'main', url: 'https://github.com/e032372/archivista-integration.git'
            }
        }
        
        stage('Build and Attest') {
            agent {
                dockerfile {
                    filename 'Dockerfile.witness'
                    dir '.'
                    args "--network archivista-network -v /var/jenkins_home:/var/jenkins_home"
                }
            }
            steps {
                sh 'openssl pkey -in /var/jenkins_home/credentials/witness-key -pubout > buildpub.pem'
                script {
                    withCredentials([file(credentialsId: 'witness-key', variable: 'KEY_FILE')]) {
                        sh """
                        witness run \\
                            --enable-archivista \\
                            --archivista-server http://archivista:8080 \\
                            --signer-file-key-path \$KEY_FILE \\
                            --step-name "build-and-attest" -- \\
                            bash -c "echo 'hello world' > hello.txt"
                        """
                    }
                }
            }
        }
    }
}
