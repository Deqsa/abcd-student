pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
    }

    stages {
        stage('Code checkout from GitHub') {
            steps {
                script {
                    cleanWs()
                    git credentialsId: 'github-token', url: 'https://github.com/Deqsa/abcd-student', branch: 'main'
                }
            }
        }

       stage('Run JuiceShop'){
            steps{
                sh 'docker run -d --rm --name juice-shop -p 3000:3000 bkimminich/juice-shop' 
            }
        }
        stage('Run ZAP DAST Scan'){
            steps{
                sh """
                    docker run --rm \
                    --add-host=host.docker.internal:host-gateway \
                    -v /var/lib/docker/volumes/abcd-lab/_data/workspace/ABCD:/zap/wrk \
                    zaproxy/zap-stable \
                    bash -c "\
                        zap.sh -cmd -addonupdate; \
                        zap.sh -cmd -addoninstall communityScripts \
                        -addoninstall pscanrulesAlpha \
                        -addoninstall pscanrulesBeta \
                        -autorun /zap/wrk/.zap/passive.yaml" 
                    """
            }
        }
    }
    post {
        always {
            echo "Cleaning up..."
            sh "docker container stop juice-shop || true"
            sh "docker container rm juice-shop || true"
            sh 'ls -la reports'
        }
        success {
            archiveArtifacts artifacts: 'reports/**/*.*', fingerprint: true
        }
    }
}
