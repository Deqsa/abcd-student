pipeline {
    agent any
    options {
        skipDefaultCheckout(true)
    }

    stages {
        stage('Code checkout from GitHub') {
            steps {
                cleanWs()
                // Upewnij się, że gałąź jest poprawna (np. 'main' lub 'master')
                git credentialsId: 'github-token', url: 'https://github.com/Deqsa/abcd-student', branch: 'main'
            }
        }

   
        stage('[ZAP] Baseline passive-scan') {
            steps {
                script {
                    // Uruchomienie juice-shop jako cel testowy
                    sh '''
                        docker run --name juice-shop -d --rm \
                            -p 3000:3000 \
                            bkimminich/juice-shop
                        sleep 5
                    '''

                    // Uruchomienie ZAP i wykorzystanie passive.yaml z repozytorium
                    sh """
                        docker run --name zap \
                            --add-host=host.docker.internal:host-gateway \
                            -v "$WORKSPACE/.zap/":/zap/.zap/:ro \
                            -t ghcr.io/zaproxy/zaproxy:stable bash -c \
                            "zap.sh -cmd -addonupdate && \
                             zap.sh -cmd -addoninstall communityScripts && \
                             zap.sh -cmd -addoninstall pscanrulesAlpha && \
                             zap.sh -cmd -addoninstall pscanrulesBeta && \
                             zap.sh -cmd -autorun /zap/.zap/passive.yaml" \
                            || true
                    """
                }
            }
        }
    }
    

    post {
        always {
            sh '''
                        docker stop zap juice-shop
                        docker rm zap
                    '''
            archiveArtifacts artifacts: 'reports/**/*.*', fingerprint: true
        }
    }
}

