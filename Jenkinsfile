pipeline {
    agent any

    environment {
        ZAP_CONFIG_DIR = "${WORKSPACE}/zap-config" // Katalog na konfigurację ZAP i wygenerowane raporty
        JUICE_SHOP_URL = "http://host.docker.internal:3000" // Cel dla ZAP
        JUICE_SHOP_CONTAINER_NAME = "juice-shop-ci" // Nazwa kontenera Juice Shop
    }

    stages {
        stage('Code checkout from GitHub') {
            steps {
                cleanWs()
                // Upewnij się, że gałąź jest poprawna (np. 'main' lub 'master')
                git credentialsId: 'github-token', url: 'https://github.com/Deqsa/abcd-student', branch: 'main'
            }
        }

        stage('Start JuiceShop') {
            steps {
                script {
                    // Zatrzymaj poprzednią instancję, jeśli istnieje
                    sh "docker ps -q --filter name=${JUICE_SHOP_CONTAINER_NAME} | grep -q . && docker stop ${JUICE_SHOP_CONTAINER_NAME} || echo 'Nie znaleziono poprzedniego kontenera ${JUICE_SHOP_CONTAINER_NAME} do zatrzymania.'"
                    
                    sh """
                        echo "Uruchamianie kontenera Juice Shop o nazwie: ${JUICE_SHOP_CONTAINER_NAME}..."
                        docker run -d --rm --name ${JUICE_SHOP_CONTAINER_NAME} -p 3000:3000 bkimminich/juice-shop
                        
                        echo "Oczekiwanie na uruchomienie Juice Shop (30 sekund)..."
                        sleep 30 // Proste oczekiwanie. Dla większej niezawodności, rozważ pętlę z curl.

                        // Informacyjne sprawdzenie dostępności Juice Shop z agenta Jenkinsa
                        echo "Próba wykonania curl na ${JUICE_SHOP_URL} z agenta Jenkinsa (tylko informacyjnie)..."
                        # Ta komenda jest wykonywana na agencie Jenkinsa. host.docker.internal może tu nie działać.
                        # ZAP w kontenerze otrzyma ten adres poprzez --add-host.
                        curl -s -o /dev/null -w '%{http_code}' ${JUICE_SHOP_URL} || echo "Ostrzeżenie: Juice Shop może nie być bezpośrednio dostępny z agenta Jenkinsa pod adresem ${JUICE_SHOP_URL}."
                        echo "Sprawdzenie zakończone."
                    """
                }
            }
        }

        stage('Prepare ZAP config') {
            steps {
                script {
                    sh '''
                        echo "Przygotowywanie konfiguracji ZAP w katalogu ${ZAP_CONFIG_DIR}..."
                        mkdir -p ${ZAP_CONFIG_DIR}
                        
                        # Sprawdzenie, czy pliki źródłowe istnieją w obszarze roboczym
                        if [ ! -f ".zap/passive.yaml" ]; then 
                            echo "BŁĄD KRYTYCZNY: Plik .zap/passive.yaml nie został znaleziony w obszarze roboczym!"
                            exit 1
                        fi
                        if [ ! -f ".zap/rules.tsv" ]; then 
                            echo "BŁĄD KRYTYCZNY: Plik .zap/rules.tsv nie został znaleziony w obszarze roboczym!"
                            exit 1
                        fi
                        
                        echo "Kopiowanie plików konfiguracyjnych ZAP..."
                        cp .zap/passive.yaml ${ZAP_CONFIG_DIR}/passive.yaml
                        cp .zap/rules.tsv ${ZAP_CONFIG_DIR}/rules.tsv
                        
                        # Nadanie uprawnień odczytu/zapisu, aby użytkownik 'zap' w kontenerze mógł pisać raporty
                        chmod -R a+rw ${ZAP_CONFIG_DIR} 
                        
                        echo "Zawartość katalogu konfiguracyjnego ZAP (${ZAP_CONFIG_DIR}):"
                        ls -l ${ZAP_CONFIG_DIR}
                        echo "Przygotowanie konfiguracji ZAP zakończone."
                    '''
                }
            }
        }

        stage('Run ZAP DAST Scan') {
            steps {
                script {
                    sh """
                        echo "Sprawdzanie pliku planu ZAP przed uruchomieniem..."
                        if [ ! -f "${ZAP_CONFIG_DIR}/passive.yaml" ]; then
                            echo "BŁĄD KRYTYCZNY: Plik ${ZAP_CONFIG_DIR}/passive.yaml nie istnieje przed uruchomieniem skanu ZAP!"
                            exit 1
                        fi

                        echo "Uruchamianie skanu ZAP... Cel: ${JUICE_SHOP_URL}"
                        echo "Plan ZAP: ${ZAP_CONFIG_DIR}/passive.yaml będzie dostępny jako /zap/wrk/passive.yaml w kontenerze."
                        echo "Raporty z ZAP powinny być zapisane w /zap/wrk/ wewnątrz kontenera (co odpowiada ${ZAP_CONFIG_DIR} na hoście)."

                        # Montowanie katalogu ZAP_CONFIG_DIR jako /zap/wrk z uprawnieniami odczytu/zapisu (rw),
                        # ponieważ passive.yaml instruuje ZAP do zapisu raportów w tym samym katalogu /zap/wrk.
                        # Uruchomienie kontenera jako użytkownik 'zap'.
                        docker run --rm \\
                            --add-host=host.docker.internal:host-gateway \\
                            -v "${ZAP_CONFIG_DIR}":"/zap/wrk":rw \\
                            -u zap \\
                            ghcr.io/zaproxy/zaproxy:stable \\
                            zap.sh -cmd -addonupdate \\
                            -addoninstall communityScripts \\
                            -addoninstall pscanrulesAlpha \\
                            -addoninstall pscanrulesBeta \\
                            -autorun /zap/wrk/passive.yaml
                        
                        zap_exit_code=\$?
                        if [ \$zap_exit_code -ne 0 ]; then
                            echo "Skan ZAP zakończony z kodem wyjścia \$zap_exit_code (wystąpiły błędy)."
                            # Możesz opcjonalnie oznaczyć build jako nieudany:
                            # currentBuild.result = 'FAILURE'
                            # error "Skan ZAP nie powiódł się z kodem wyjścia \$zap_exit_code"
                        else
                            echo "Skan ZAP zakończony pomyślnie."
                        fi
                        
                        echo "Zawartość katalogu ${ZAP_CONFIG_DIR} po skanie ZAP:"
                        ls -lA ${ZAP_CONFIG_DIR}
                    """
                }
            }
        }

        stage('Archive ZAP Reports') {
            steps {
                // Raporty są oczekiwane w ZAP_CONFIG_DIR, zgodnie z montowaniem i konfiguracją passive.yaml
                echo "Archiwizowanie raportów ZAP z katalogu ${ZAP_CONFIG_DIR}..."
                archiveArtifacts artifacts: "${ZAP_CONFIG_DIR}/*.html, ${ZAP_CONFIG_DIR}/*.xml, ${ZAP_CONFIG_DIR}/*.json", \
                                 allowEmptyArchive: true, \
                                 fingerprint: true // Opcjonalnie, dodaje fingerprint do artefaktów
                echo "Archiwizacja raportów ZAP zakończona."
            }
        }
    }

    post {
        always {
            script {
                echo 'Wykonywanie akcji czyszczących po buildzie...'
                // Kontener Juice Shop jest uruchamiany z --rm, więc zostanie automatycznie usunięty po zatrzymaniu.
                // Wystarczy 'docker stop'.
                sh "echo 'Zatrzyywanie kontenera ${JUICE_SHOP_CONTAINER_NAME}...' && docker stop ${JUICE_SHOP_CONTAINER_NAME} || echo 'Kontener ${JUICE_SHOP_CONTAINER_NAME} już zatrzymany lub nie znaleziony.'"
                // cleanWs() // Odkomentuj, jeśli chcesz czyścić obszar roboczy po każdym uruchomieniu
                echo "Czyszczenie zakończone."
            }
        }
        failure {
            echo 'Pipeline zakończony niepowodzeniem.'
        }
        success {
            echo 'Pipeline zakończony pomyślnie.'
        }
    }
}
