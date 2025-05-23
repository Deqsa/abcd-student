pipeline {
    agent any

    environment {
        ZAP_CONTAINER = 'zap'
        JUICE_CONTAINER = 'juice-shop'
    }

    stages {
        stage('Code checkout from GitHub') {
            steps {
                cleanWs()
                git branch: 'main', credentialsId: 'github-token', url: 'https://github.com/Deqsa/abcd-student'
            }
        }

        stage('Run JuiceShop') {
            steps {
                sh '''
                    docker run -d --rm --name $JUICE_CONTAINER -p 3000:3000 bkimminich/juice-shop
                    sleep 10
                '''
            }
        }

        stage('Run ZAP DAST Scan') {
            steps {
                sh '''
                    docker run --rm \
                        --add-host=host.docker.internal:host-gateway \
                        -v "$WORKSPACE/.zap:/zap/wrk" \
                        ghcr.io/zaproxy/zaproxy:stable \
                        zap.sh -cmd \
                        -addonupdate \
                        -addoninstall communityScripts \
                        -addoninstall pscanrulesAlpha \
                        -addoninstall pscanrulesBeta \
                        -autorun /zap/wrk/passive.yaml \
                        || true
                '''
            }
        }
    }

    post {
        always {
            echo 'Cleaning up...'
            sh '''
                docker container stop $JUICE_CONTAINER || true
                docker container rm $JUICE_CONTAINER || true
                mkdir -p reports
                cp -r .zap/reports/* reports/ || true
            '''
            archiveArtifacts artifacts: 'reports/**/*', allowEmptyArchive: true
        }
    }
}
