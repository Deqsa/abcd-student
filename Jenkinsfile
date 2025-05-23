pipeline {
    agent any

    environment {
        ZAP_CONFIG_DIR = "${WORKSPACE}/zap-config"
    }

    stages {
        stage('Code checkout from GitHub') {
            steps {
                cleanWs()
                git credentialsId: 'github-token', url: 'https://github.com/Deqsa/abcd-student', branch: 'main'
            }
        }

        stage('Run JuiceShop') {
            steps {
                sh '''
                    docker run -d --rm --name juice-shop -p 3000:3000 bkimminich/juice-shop
                    sleep 10
                '''
            }
        }

        stage('Prepare ZAP config') {
            steps {
                sh '''
                    echo "Preparing ZAP config..."
                    mkdir -p ${ZAP_CONFIG_DIR}
                    cp .zap/passive.yaml ${ZAP_CONFIG_DIR}/passive.yaml
                    cp .zap/rules.tsv ${ZAP_CONFIG_DIR}/rules.tsv || true

                    echo "Contents of ${ZAP_CONFIG_DIR}:"
                    ls -l ${ZAP_CONFIG_DIR}
                '''
            }
        }

        stage('Run ZAP DAST Scan') {
            steps {
                sh '''
                    docker run --rm \
                        --add-host=host.docker.internal:host-gateway \
                        -v "${ZAP_CONFIG_DIR}:/zap/wrk:ro" \
                        ghcr.io/zaproxy/zaproxy:stable \
                        zap.sh -cmd -addonupdate -addoninstall communityScripts -addoninstall pscanrulesAlpha -addoninstall pscanrulesBeta -autorun /zap/wrk/passive.yaml
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
            archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: true
        }
    }
}
