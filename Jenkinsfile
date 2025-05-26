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

   stage('Prepare test environment') {
            steps {
                sh 'mkdir reports'
                sh 'chmod 777 reports'
                sh '''
                    if [ $(docker ps -q -f name=juice-shop) ]; then
                        echo "Stopping existing juice-shop container"
                        docker stop juice-shop || true
                    fi
                    '''
                sh '''
                    if [ $(docker ps -a -q -f name=zap) ]; then
                        echo "Stopping and removing existing zap container"
                        docker stop zap || true
                        docker rm zap || true
                    fi
                    '''
            }
        }
        stage('[OSV] Scan package-lock.json file') {
            steps {
                sh 'osv-scanner --format json --output reports/osv_json_report.json --lockfile package-lock.json || true'
            }
        }
        stage('[ZAP] Baseline passive-scan') {
            steps {
                sh 'mkdir -p results/'
                sh '''
                    docker run --name juice-shop -d --rm \
                        -p 3000:3000 \
                        bkimminich/juice-shop
                    sleep 5
                '''
                sh '''
                    docker run --name zap \
                        --add-host=host.docker.internal:host-gateway \
                        -v /var/lib/docker/volumes/abcd-lab/_data/workspace/ABCD/:/zap/wrk/:rw \
                        -t ghcr.io/zaproxy/zaproxy:stable bash -c \
                        "zap.sh -cmd -addonupdate; zap.sh -cmd -addoninstall communityScripts -addoninstall pscanrulesAlpha -addoninstall pscanrulesBeta -autorun /zap/wrk/.zap/passive.yaml" \
                        || true
                '''
            }
        }
    }
    post {
        always {
            sh '''
                        docker stop zap juice-shop
                        docker rm zap
                    '''
            archiveArtifacts artifacts: 'reports/**/*.*', fingerprint: true
        }
    }
}
