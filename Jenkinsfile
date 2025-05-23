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
                script {
                    sh 'docker run -d --rm --name juice-shop -p 3000:3000 bkimminich/juice-shop'
                    sleep 10
                }
            }
        }

        stage('Prepare ZAP config') {
            steps {
                script {
                    sh '''
                        echo Preparing ZAP config...
                        mkdir -p ${ZAP_CONFIG_DIR}
                        cp .zap/passive.yaml ${ZAP_CONFIG_DIR}/passive.yaml
                        cp .zap/rules.tsv ${ZAP_CONFIG_DIR}/rules.tsv
                        echo Zawartość katalogu ZAP config:
                        ls -l ${ZAP_CONFIG_DIR}
                    '''
                }
            }
        }

        stage('Run ZAP DAST Scan') {
            steps {
                script {
                    sh '''
                        echo Running ZAP scan...
                        [ ! -f ${ZAP_CONFIG_DIR}/passive.yaml ] && echo "Missing passive.yaml!" && exit 1
                        docker run --rm \
                            --add-host=host.docker.internal:host-gateway \
                            -v ${ZAP_CONFIG_DIR}:/zap/wrk \
                            ghcr.io/zaproxy/zaproxy:stable \
                            zap.sh -cmd -addonupdate \
                            -addoninstall communityScripts \
                            -addoninstall pscanrulesAlpha \
                            -addoninstall pscanrulesBeta \
                            -autorun /zap/wrk/passive.yaml
                    '''
                }
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
                [ -d .zap/reports ] && cp -r .zap/reports/* reports/
            '''
            archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: true
        }
    }
}
