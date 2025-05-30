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
            sh '''
                docker run --name juice-shop -d --rm -p 3000:3000 bkimminich/juice-shop
                sleep 5
            '''

            sh 'mkdir -p zap-output/reports' // <- DODANE

            sh """
                docker run --name zap --rm \
                    --add-host=host.docker.internal:host-gateway \
                    -v "${env.WORKSPACE}/.zap/passive.yaml:/zap/.zap/passive.yaml:ro" \
                    -v "${env.WORKSPACE}/zap-output:/zap/wrk:rw" \
                    -t ghcr.io/zaproxy/zaproxy:stable bash -c \
                    "zap.sh -cmd -addonupdate && \
                     zap.sh -cmd -addoninstall communityScripts && \
                     zap.sh -cmd -addoninstall pscanrulesAlpha && \
                     zap.sh -cmd -addoninstall pscanrulesBeta && \
                     zap.sh -cmd -autorun /zap/.zap/passive.yaml"
            """
        }
    }
}
    
    

post {
    always {
        sh '''
            docker stop juice-shop || true
        '''
        archiveArtifacts artifacts: 'zap-output/reports/**/*.*', fingerprint: true
    }
}

