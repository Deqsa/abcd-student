pipeline {
    agent any

    environment {
        ZAP_CONFIG_DIR = "${WORKSPACE}/zap-config"
        ZAP_AUTORUN_FILE = "${ZAP_CONFIG_DIR}/passive.yaml"
        ZAP_RULES_FILE = "${ZAP_CONFIG_DIR}/rules.tsv"
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
                script {
                    sh 'docker run -d --rm --name juice-shop -p 3000:3000 bkimminich/juice-shop'
                    sleep(time: 10, unit: 'SECONDS')
                }
            }
        }

        stage('Prepare ZAP config') {
            steps {
                script {
                    sh '''
                        echo "Preparing ZAP config..."
                        mkdir -p ${ZAP_CONFIG_DIR}/autorun
                        cp .zap/passive.yaml ${ZAP_CONFIG_DIR}/autorun/passive.yaml
                        cp .zap/rules.tsv ${ZAP_CONFIG_DIR}/rules.tsv
                        echo "Zawartość katalogu ZAP config:"
                        ls -l ${ZAP_CONFIG_DIR}/autorun
                    '''
                }
            }
        }

        stage('Run ZAP DAST Scan') {
            steps {
                script {
                    sh '''
                        echo "Running ZAP scan..."
                        if [ ! -f "${ZAP_CONFIG_DIR}/autorun/passive.yaml" ]; then
                            echo "ERROR: passive.yaml not found!"
                            exit 1
                        fi

                        docker run --rm \
                          --add-host=host.docker.internal:host-gateway \
                          -v ${ZAP_CONFIG_DIR}:/zap/wrk \
                          ghcr.io/zaproxy/zaproxy:stable \
                          zap.sh -cmd -addonupdate \
                          -addoninstall communityScripts \
                          -addoninstall pscanrulesAlpha \
                          -addoninstall pscanrulesBeta \
                          -autorun /zap/wrk/autorun/passive.yaml
                    '''
                }
            }
        }
    }

    post {
        always {
            echo 'Cleaning up...'

            // Stop and remove Juice Shop
            sh '''
                docker container stop juice-shop || true
                docker container rm juice-shop || true
            '''

            // Collect any ZAP reports
            sh '''
                mkdir -p reports
                if [ -d .zap/reports ]; then
                    cp -r .zap/reports/* reports/ || true
                fi
            '''

            // Archive results
            archiveArtifacts artifacts: 'reports/**/*', onlyIfSuccessful: false
        }
    }
}
