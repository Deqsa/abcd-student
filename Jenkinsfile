pipeline {
    agent any
    options {
        skipDefaultCheckout(true)
    }
    stages {
        stage('[ZAP] Baseline passive-scan') {
    steps {
        script {
            // Stwórz katalog na raporty
            sh 'mkdir -p ${WORKSPACE}/results'

            // Uruchom aplikację Juice Shop
            sh '''
                docker run -d --name juice-shop \
                    -p 3000:3000 \
                    bkimminich/juice-shop
                sleep 10
            '''

            // Uruchom skan ZAP Automation Framework
            sh '''
                docker run --name zap \
                    --add-host=host.docker.internal:host-gateway \
                    -v ${WORKSPACE}/zap-config:/zap/wrk/:rw \
                    ghcr.io/zaproxy/zaproxy:stable \
                    bash -c "zap.sh -cmd -addonupdate && \
                             zap.sh -cmd -addoninstall communityScripts && \
                             zap.sh -cmd -addoninstall pscanrulesAlpha && \
                             zap.sh -cmd -addoninstall pscanrulesBeta && \
                             zap.sh -cmd -autorun /zap/wrk/passive_scan.yaml" || true
            '''

            // Skopiuj raporty z kontenera do katalogu Jenkins
            sh '''
                docker cp zap:/zap/wrk/reports/zap_html_report.html ${WORKSPACE}/results/zap_html_report.html || true
                docker cp zap:/zap/wrk/reports/zap_xml_report.xml ${WORKSPACE}/results/zap_xml_report.xml || true
                docker stop zap || true
                docker stop juice-shop || true
            '''
        }
    }
    post {
        always {
            archiveArtifacts artifacts: 'results/*.html, results/*.xml', allowEmptyArchive: true
        }
    }
}
    }
}
