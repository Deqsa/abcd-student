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

        stage('Debug .zap directory') {
            steps {
                echo 'Listing .zap directory contents:'
                sh 'ls -l .zap'
                echo 'Listing passive.yaml file details:'
                sh 'ls -l .zap/passive.yaml'
            }
        }

        stage('Run ZAP DAST Scan') {
            steps {
                sh '''
                docker run --rm --add-host=host.docker.internal:host-gateway \
                    -v ${WORKSPACE}/.zap:/zap/wrk:ro \
                    ghcr.io/zaproxy/zaproxy:stable \
                    zap.sh -cmd -addonupdate -addoninstall communityScripts \
                    -addoninstall pscanrulesAlpha -addoninstall pscanrulesBeta \
                    -autorun /zap/wrk/passive.yaml
                '''
            }
        }
    }

    post {
        always {
            echo 'Cleaning up...'
            sh '''
                docker container stop juice-shop || true
                docker container rm juice-shop || true
                mkdir -p reports
                cp -r .zap/reports/* reports/ || true
            '''
            archiveArtifacts artifacts: 'reports/**/*', allowEmptyArchive: true
        }
    }
}
