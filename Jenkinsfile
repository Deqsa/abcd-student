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

        stage('[ZAP] Baseline passive-scan') {
            steps {
                script {
                    sh 'mkdir -p results/'

                    // Upewniamy się, że nie ma kolidujących kontenerów
                    sh 'docker rm -f juice-shop || true'
                    sh 'docker rm -f zap || true'

                    // Uruchamiamy aplikację Juice Shop
                    sh '''
                        docker run -d --name juice-shop \
                            -p 3000:3000 \
                            bkimminich/juice-shop
                        sleep 10
                    '''

                    // Uruchamiamy ZAP z konfiguracją z .zap/passive.yaml
                    sh '''
                        docker run --name zap \
                            --add-host=host.docker.internal:host-gateway \
                            -v "$PWD/.zap:/zap/wrk/:rw" \
                            ghcr.io/zaproxy/zaproxy:stable \
                            zap.sh -cmd -addonupdate -addoninstall communityScripts -addoninstall pscanrulesAlpha -addoninstall pscanrulesBeta \
                            -autorun /zap/wrk/passive.yaml || true
                    '''
                }
            }

            post {
                always {
                    script {
                        // Zgrywamy raporty
                        sh '''
                            docker cp zap:/zap/wrk/reports/zap_html_report.html results/zap_html_report.html || true
                            docker cp zap:/zap/wrk/reports/zap_xml_report.xml results/zap_xml_report.xml || true
                        '''

                        // Sprzątamy kontenery
                        sh 'docker rm -f zap || true'
                        sh 'docker rm -f juice-shop || true'
                    }

                    // Archiwizujemy raporty
                    archiveArtifacts artifacts: 'results/*.html, results/*.xml', allowEmptyArchive: true
                }
            }
        }
    }
}
