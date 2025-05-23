pipeline {
    agent any

    environment {
        ZAP_CONFIG_DIR = "${WORKSPACE}/zap-config"
        ZAP_REPORTS_DIR = "${WORKSPACE}/zap-reports"
        // Użyj host.docker.internal dla Docker Desktop (Windows/Mac) aby kontener ZAP mógł uzyskać dostęp do Juice Shop na hoście.
        // Dla Linuxa, jeśli Jenkins i ZAP działają w Dockerze, może być konieczne użycie adresu IP hosta
        // lub konfiguracja wspólnej sieci Docker.
        JUICE_SHOP_URL = "http://host.docker.internal:3000"
        JUICE_SHOP_CONTAINER_NAME = "juice-shop-ci" // Unikalna nazwa, aby uniknąć konfliktów
    }

    stages {
        stage('Code checkout from GitHub') {
            steps {
                cleanWs()
                // Upewnij się, że 'github-token' to poprawny ID Twoich danych uwierzytelniających GitHub w Jenkinsie
                git credentialsId: 'github-token', url: 'https://github.com/Deqsa/abcd-student', branch: 'main'
            }
        }

        stage('Start JuiceShop') {
            steps {
                script {
                    // Zatrzymaj i usuń poprzedni kontener Juice Shop, jeśli istnieje, aby uniknąć konfliktów portów
                    sh "docker ps -q --filter name=${JUICE_SHOP_CONTAINER_NAME} | grep -q . && docker stop ${JUICE_SHOP_CONTAINER_NAME} || echo 'No previous JuiceShop container to stop.'"
                    // Uruchom Juice Shop w tle, usuń po zatrzymaniu, nadaj nazwę i opublikuj port
                    sh "docker run -d --rm --name ${JUICE_SHOP_CONTAINER_NAME} -p 3000:3000 bkimminich/juice-shop"
                    
                    // Poczekaj aż Juice Shop się uruchomi.
                    // Dla pewności, można zaimplementować pętlę sprawdzającą dostępność URL.
                    sh 'echo "Waiting for Juice Shop to start (30 seconds)..." && sleep 30'
                    
                    // Sprawdzenie czy Juice Shop odpowiada (opcjonalne, ale zalecane)
                    sh "curl -s -o /dev/null -w '%{http_code}' ${JUICE_SHOP_URL} || echo 'Juice Shop might not be accessible at ${JUICE_SHOP_URL}'"
                }
            }
        }

        stage('Prepare ZAP Configuration') {
            steps {
                script {
                    sh '''
                        echo "Preparing ZAP Configuration..."
                        mkdir -p ${ZAP_CONFIG_DIR}
                        mkdir -p ${ZAP_REPORTS_DIR}
                        # Kopiowanie konfiguracji pasywnego skanu ZAP
                        # Zakładamy, że plik passive.yaml znajduje się w katalogu .zap w Twoim repozytorium
                        if [ -f ".zap/passive.yaml" ]; then
                            cp .zap/passive.yaml ${ZAP_CONFIG_DIR}/passive.yaml
                            echo "Copied .zap/passive.yaml to ${ZAP_CONFIG_DIR}"
                            # (Opcjonalnie) Możesz tutaj zmodyfikować passive.yaml, np. wstawić dynamicznie JUICE_SHOP_URL
                            # sed -i "s|%%TARGET_URL%%|${JUICE_SHOP_URL}|g" ${ZAP_CONFIG_DIR}/passive.yaml
                        else
                            echo "ERROR: .zap/passive.yaml not found!"
                            exit 1
                        fi
                    '''
                }
            }
        }

        stage('Run ZAP Passive Scan') {
            steps {
                script {
                    sh """
                        echo "Running ZAP Passive Scan against ${JUICE_SHOP_URL}"
                        # Użyj --add-host, aby ZAP w kontenerze mógł rozwiązać host.docker.internal (dla Docker Desktop)
                        # Dla Linuxa, może być konieczne użycie --network="host" lub IP hosta w JUICE_SHOP_URL
                        # Twój plik passive.yaml musi być planem ZAP Automation Framework.
                        # Upewnij się, że passive.yaml:
                        # 1. Ma ustawiony prawidłowy cel skanowania (np. przez zmienną lub zahardkodowany ${JUICE_SHOP_URL}).
                        #    Jeśli passive.yaml ma zahardkodowany URL, upewnij się, że jest to ${JUICE_SHOP_URL}.
                        #    Można go nadpisać za pomocą parametrów -J dla ZAP Automation Framework, np.:
                        #    -J env.contexts[0].urls[0]=${JUICE_SHOP_URL}
                        # 2. Generuje raport do katalogu /zap/wrk/ (np. jako report.html).

                        docker run --add-host=host.docker.internal:host-gateway --rm \\
                            -v "${ZAP_CONFIG_DIR}":/zap/config:ro \\
                            -v "${ZAP_REPORTS_DIR}":/zap/wrk/:rw \\
                            -t owasp/zap2docker-stable zap.sh -cmd -autorun /zap/config/passive.yaml
                        
                        echo "ZAP Scan finished. Reports should be in ${ZAP_REPORTS_DIR}"
                    """
                }
            }
        }

        stage('Publish ZAP Report') {
            steps {
                // Archiwizuj wszystkie wygenerowane raporty HTML, XML, JSON
                archiveArtifacts artifacts: "${ZAP_REPORTS_DIR}/*.html, ${ZAP_REPORTS_DIR}/*.xml, ${ZAP_REPORTS_DIR}/*.json", followSymlinks: false, allowEmptyArchive: true
                echo "ZAP reports archived."
            }
        }
    }

post {
        always {
            // Bezpośrednio kroki, bez dodatkowego 'stage'
            script {
                echo 'Executing post-build cleanup actions...'
                // Zawsze próbuj zatrzymać kontener Juice Shop
                sh "echo 'Stopping JuiceShop container (${JUICE_SHOP_CONTAINER_NAME})...' && docker stop ${JUICE_SHOP_CONTAINER_NAME} || echo 'JuiceShop container (${JUICE_SHOP_CONTAINER_NAME}) not found or already stopped.'"
            }
            // Clean up workspace
            cleanWs()
            echo 'Workspace cleaned up.'
        }
    }
}
