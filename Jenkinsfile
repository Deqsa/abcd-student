pipeline {More actions
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
                         docker run --name zap --rm \
        --add-host=host.docker.internal:host-gateway \
        -v "/mnt/c/Users/Don/Documents/GitHub/abcd-student/.zap/passive.yaml:/zap/.zap/passive.yaml:ro" \
        -v "/mnt/c/Users/Don/zap-output:/zap/wrk" \
        -t ghcr.io/zaproxy/zaproxy:stable bash -c \
        "mkdir -p /zap/wrk/reports && \
         zap.sh -cmd -addonupdate && \
         zap.sh -cmd -addoninstall communityScripts && \
         zap.sh -cmd -addoninstall pscanrulesAlpha && \
         zap.sh -cmd -addoninstall pscanrulesBeta && \
         zap.sh -cmd -autorun /zap/.zap/passive.yaml"
                    """
                }
            }
        }
    }



post {
    always {
        script {
            sh 'docker stop juice-shop || true'
            sh 'docker stop zap || true'
            sh 'docker rm zap || true'
        }
        archiveArtifacts artifacts: 'reports/**/*.*', fingerprint: true
    }
}
}
