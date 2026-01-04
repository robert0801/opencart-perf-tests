pipeline {
    agent any

    parameters {
        choice(name: 'MODE', choices: ['guest', 'mix'], description: 'Сценарий: только гость или 50/50')
        choice(name: 'LOAD_TYPE', choices: ['By Duration (по времени)', 'By Loops (по кругам)'], description: 'Как ограничиваем тест?')
        string(name: 'THREADS', defaultValue: '1', description: 'Количество пользователей')
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

                    sh """
                        ${JM_BIN} -n -t ${JMX_FILE} -l ${RESULTS_FILE} \
                        -Jmode=${params.MODE} \
                        -Jthreads=${params.THREADS} \
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
                // archiveArtifacts работает напрямую в steps, script тут не обязателен
                archiveArtifacts artifacts: "${RESULTS_FILE}", fingerprint: true, allowEmptyArchive: true
            }
        }
    } // Конец stages

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