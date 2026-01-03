pipeline {
    agent any

        parameters {
            string(name: 'THREADS', defaultValue: '1', description: 'Количество потоков')
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
                        echo "--- JMeter already installed in ${JM_HOME} ---"
                    }
                }
            }
        }

        stage('Run Load Test') {
            steps {
                echo "--- Starting JMeter Load Test ---"
                sh ${JM_BIN} -n -t ${JMX_FILE} -l ${RESULTS_FILE} -Jthreads=${params.THREADS}
            }
        }

        stage('Archive Results') {
            steps {
                archiveArtifacts artifacts: "${RESULTS_FILE}", fingerprint: true
            }
        }
    }

    post {
        always {
            echo "--- Test Completed. Check Grafana for real-time metrics ---"
            // Если установлен Performance Plugin, он нарисует графики прямо в Jenkins
            perfReport sourceDataFiles: "**/${RESULTS_FILE}"
        }
        failure {
            echo "--- Test Failed! ---"
        }
    }
}