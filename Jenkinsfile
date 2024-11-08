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
                    if (env.targetEnv == "blue") {
                        env.otherEnv = "green"
                    } else if (env.targetEnv == "green") {
                        env.otherEnv = "blue"
                    }
                }
            }
        }
        stage('deploy new version') {
            steps {
                sh "helm upgrade demo-app-${env.targetEnv} ./app-chart --namespace ${env.namespace} -f ./values-${env.targetEnv}.yaml"
                sh "sleep 10"
            }
        }
        stage('test new version') {
            steps {
                script {
                    def statusCode = sh(script: "curl -o /dev/null -s -w '%{http_code}' http://demo-app-${env.targetEnv}.blue-green.svc.cluster.local:5000/version", returnStdout: true).trim()
                    echo "test response status code: ${statusCode}"
                    if (statusCode == '200') {
                        env.testing = "passed"
                    } else {
                        echo "test run failed: ${statusCode}"
                        env.testing = "failed"
                    }
                    echo "test ${env.testing}"
                    echo "env switching is set to ${env.switchEnvs}"
                }
            }
        }
        stage('Switch Router') {
            when {
                expression { env.testing == "passed" && env.switchEnvs == "true" }
            }
            steps {
                sh "echo switching"
                sh "helm upgrade blue-green-ingress ./ingress-chart --namespace ${env.namespace} --set ingress.target=demo-app-${env.targetEnv}"
                sh "echo switched app environments, now ${env.targetEnv} is active"
            }
        }
        stage('live environment test') {
            steps {
                script {
                    def statusCode = sh(script: "curl -o /dev/null -s -w '%{http_code}' ${env.testEndpoint}", returnStdout: true).trim()
                    echo "live environment test response status code: ${statusCode}"
                    if (statusCode == '200') {
                        env.testing = "passed"
                        echo "live environment test passed"
                    } else {
                        echo "test run failed: ${statusCode}"
                    }
                    echo "test ${env.testing}"
                    echo "env switching is set to ${env.switchEnvs}"
                }
            }
        }
        stage('rollback') {
            when {
                expression { env.testing != "passed" && env.switchEnvs == "true" }
            }
            steps {
                sh "echo switching back due to failed test"
                sh "helm upgrade blue-green-ingress --namespace ${env.namespace} --set ingress.target=demo-app-${env.otherEnv}"
                sh "echo switched back app environments, now ${env.otherEnv} is active"
            }
        }
        stage('scaledown') {
            when {
                expression { env.testing == "passed" && env.switchEnvs == "true" && env.scaleDown == "true" }
            }
            steps {
                sh "echo scaling down old version"
                sh "helm upgrade demo-app-${env.otherEnv} ./app-chart --namespace ${env.namespace} -f ./values-${env.otherEnv}.yaml --set replicaCount=0"
                sh "echo scaled down ${env.otherEnv} environment"
            }
        }
    }
}
