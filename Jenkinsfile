pipeline {
    agent any

    parameters {
        // Выбор сценария
        choice(name: 'TARGET',
               choices: ['e2e', 'home', 'login', 'entries', 'addtocart', 'viewcart', 'view', 'deleteitem'],
               description: 'Что тестируем? e2e - весь сценарий, остальное - изоляция эндпоинтов')

        choice(name: 'MODE', choices: ['guest', 'mix'], description: 'Сценарий: только гость или 50/50')
        choice(name: 'LOAD_TYPE', choices: ['By Duration (по времени)', 'By Loops (по кругам)'], description: 'Как ограничиваем тест?')

        string(name: 'THREADS', defaultValue: '1', description: 'Количество пользователей для активного таргета')
        string(name: 'RAMPUP', defaultValue: '1', description: 'Разгон (сек)')
        string(name: 'DURATION_VAL', defaultValue: '60', description: 'Если выбрали "по времени" (сек)')
        string(name: 'LOOPS_VAL', defaultValue: '1', description: 'Если выбрали "по кругам" (количество повторов)')
    }

    environment {
        JM_VERSION = "5.6.3"
        JM_HOME = "/var/jenkins_home/apache-jmeter-${JM_VERSION}"
        JM_BIN  = "${JM_HOME}/bin/jmeter.sh"
        JMX_FILE = "Opencart.jmx"
        RESULTS_FILE = "results.jtl"
    }

    stages {
        stage('Install JMeter') {
            steps {
                script {
                    def exists = sh(script: "test -d ${JM_HOME}", returnStatus: true) == 0
                    if (!exists) {
                        echo "--- JMeter not found. Downloading version ${JM_VERSION} ---"
                        sh "wget -q https://archive.apache.org/dist/jmeter/binaries/apache-jmeter-${JM_VERSION}.tgz -P /tmp"
                        sh "tar -xzf /tmp/apache-jmeter-${JM_VERSION}.tgz -C /var/jenkins_home/"
                        sh "chmod +x ${JM_BIN}"
                    } else {
                        echo "--- JMeter already installed ---"
                    }
                }
            }
        }

        stage('Run Load Test') {
            steps {
                script {
                    // Инициализируем потоки всех групп нулями
                    def t_e2e = 0
                    def t_home = 0
                    def t_login = 0
                    def t_entries = 0
                    def t_addtocart = 0
                    def t_viewcart = 0
                    def t_view = 0
                    def t_deleteitem = 0

                    // Присваиваем THREADS только выбранному таргету
                    switch(params.TARGET) {
                        case 'e2e': t_e2e = params.THREADS; break
                        case 'home': t_home = params.THREADS; break
                        case 'login': t_login = params.THREADS; break
                        case 'entries': t_entries = params.THREADS; break
                        case 'addtocart': t_addtocart = params.THREADS; break
                        case 'viewcart': t_viewcart = params.THREADS; break
                        case 'view': t_view = params.THREADS; break
                        case 'deleteitem': t_deleteitem = params.THREADS; break
                    }

                    // Логика Loops vs Duration
                    def jm_loops = ""
                    def jm_duration = ""
                    if (params.LOAD_TYPE.contains('Duration')) {
                        echo "--- Режим: ТЕСТ ПО ВРЕМЕНИ (${params.DURATION_VAL} сек) ---"
                        jm_loops = "-1"
                        jm_duration = params.DURATION_VAL
                    } else {
                        echo "--- Режим: ТЕСТ ПО ЦИКЛАМ (${params.LOOPS_VAL} кругов) ---"
                        jm_loops = params.LOOPS_VAL
                        jm_duration = "999999"
                    }

                    sh "rm -f ${RESULTS_FILE}"

                    // Запуск с передачей всех специфичных переменных потоков
                    sh """
                        ${JM_BIN} -n -t ${JMX_FILE} -l ${RESULTS_FILE} \
                        -Jt_e2e=${t_e2e} \
                        -Jt_home=${t_home} \
//                         -Jt_login=${t_login} \
//                         -Jt_entries=${t_entries} \
//                         -Jt_addtocart=${t_addtocart} \
//                         -Jt_viewcart=${t_viewcart} \
//                         -Jt_view=${t_view} \
//                         -Jt_deleteitem=${t_deleteitem} \
                        -Jmode=${params.MODE} \
                        -Jrampup=${params.RAMPUP} \
                        -Jloops=${jm_loops} \
                        -Jduration=${jm_duration} \
                        -Jinflux_host=influxdb
                    """
                }
            }
        }

        stage('Archive Results') {
            steps {
                archiveArtifacts artifacts: "${RESULTS_FILE}", fingerprint: true, allowEmptyArchive: true
            }
        }
    }

    post {
        always {
            echo "--- Test Completed. Check Grafana for real-time metrics ---"
            perfReport sourceDataFiles: "**/${RESULTS_FILE}"
        }
        failure {
            echo "--- Test Failed! ---"
        }
    }
}