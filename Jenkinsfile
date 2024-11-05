pipeline {
    agent any

    stages {
        stage('clone repo') {
            steps {
                git url: 'https://github.com/bartek-babu/demo-blue-green.git', branch: 'main' 
            }
        }
        stage('Load Variables') {
            steps {
                script {
                    def props = readProperties  file: "./pipelineVars"
                    keys= props.keySet()
                    for (key in keys) {
                        def value = props["${key}"]
                        env."${key}" = "${value}"
                    }
                    if (props.targetEnv == "blue") {
                        env.otherEnv = "green"
                    } else if (props.targetEnv == "green") {
                        env.otherEnv = "blue"
                    }
                }
            }
        }
        stage('deploy new version') {
            steps {
                sh "helm upgrade demo-app-${env.targetEnv} ./app-chart --namespace ${env.namespace} -f .values-${env.targetEnv}.yaml"
            }
        }
        stage('test new version') {
            steps {
                script {
                    def statusCode = sh(script: 'curl -o /dev/null -s -w "%{http_code}" http://demo-app-${env.targetEnv}.blue-green.svc.cluster.local:5000/version', returnStdout: true).trim()
                    echo "test response status code: ${statusCode}"
                    if (statusCode == '200') {
                        env.testing = "passed"
                    } else {
                        echo "test run failed: ${statusCode}"
                    }
                }
            }
        }
        stage('Switch Router') {
            when {
                expression { env.testing == "passed" && env.switchEnvs == "true" }
            }
            steps {
                sh 'echo switching'
                sh 'helm upgrade blue-green-ingress --namespace ${env.namespace} --set target=demo-app-${env.targetEnv}'
                sh 'echo "switched app environments, now ${env.targetEnv} is active"'
            }
        }
        stage('live environment test') {
            steps {
                script {
                    def statusCode = sh(script: 'curl -o /dev/null -s -w "%{http_code}" ${env.testEndpoint}', returnStdout: true).trim()
                    echo "live environment test response status code: ${statusCode}"
                    if (statusCode == '200') {
                        env.testing = "passed"
                        echo "live environment test passed"
                    } else {
                        echo "test run failed: ${statusCode}"
                    }
                }
            }
        }
        stage('rollback') {
            when {
                expression { env.testing != "passed" && env.switchEnvs == "true" }
            }
            steps {
                sh 'echo switching back due to failed test'
                sh 'helm upgrade blue-green-ingress --namespace ${env.namespace} --set target=demo-app-${env.otherEnv}'
                sh 'echo "switched back app environments, now ${env.otherEnv} is active"'
            }
        }
    }
}
