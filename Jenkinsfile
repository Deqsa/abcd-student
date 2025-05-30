pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
    }

    stages {
        stage('Checkout Code from GitHub') {
            steps {
                cleanWs()
                git credentialsId: 'github-token', url: 'https://github.com/Deqsa/abcd-student', branch: 'main'
            }
        }
/*
        stage('[ZAP] Passive Scan') {
            steps {
                script {
                    sh '''
                        docker run --name juice-shop -d --rm \
                            -p 3000:3000 \
                            bkimminich/juice-shop
                        sleep 10
                    '''

                    sh '''
                        docker run --name zap --rm \
                            --add-host=host.docker.internal:host-gateway \
                            -v "/mnt/c/Users/Don/Documents/GitHub/abcd-student/.zap/passive.yaml:/zap/.zap/passive.yaml:ro" \
                            -v "/mnt/c/Users/Don/zap-output:/zap/wrk/reports" \
                            ghcr.io/zaproxy/zaproxy:stable bash -c '
                                mkdir -p /zap/wrk/reports && \
                                zap.sh -cmd -addonupdate && \
                                zap.sh -cmd -addoninstall communityScripts && \
                                zap.sh -cmd -addoninstall pscanrulesAlpha && \
                                zap.sh -cmd -addoninstall pscanrulesBeta && \
                                zap.sh -cmd -autorun /zap/.zap/passive.yaml
                            '
                    '''
                }
            }
        }
*/
        stage('[TruffleHog] Secret Scan') {
            steps {
                sh '''
                    mkdir -p ${WORKSPACE}/zap-output
trufflehog git --branch main --json . > ${WORKSPACE}/zap-output/trufflehog-report.json
                '''
                archiveArtifacts artifacts: '/mnt/c/Users/Don/zap-output/trufflehog-report.json', fingerprint: true
            }
        }

        stage('[Semgrep] Static Code Analysis') {
            steps {
                sh '''
                    semgrep --config p/default --json --output /mnt/c/Users/Don/zap-output/semgrep-report.json
                '''
                archiveArtifacts artifacts: '/mnt/c/Users/Don/zap-output/semgrep-report.json', fingerprint: true
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

            archiveArtifacts artifacts: '/mnt/c/Users/Don/zap-output/**/*.*', fingerprint: true
        }
    }
}
