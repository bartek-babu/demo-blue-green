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
                    for(key in keys) {
                        value = props["${key}"]
                        env."${key}" = "${value}"
                    }
                    if (props.targetEnv == "blue") {
                        props.otherEnv = "green"
                    } else if (props.targetEnv == "green") {
                        props.otherEnv = "blue"
                    }
                }
            }
        }
        stage('deploy new version') {
            steps {
                sh "helm upgrade demo-app-${props.targetEnv} --namespace ${props.namespace} -f .values-${props.targetEnv}.yaml"
            }
        }
        stage('test new version') {
            steps {
                script {
                    def statusCode = sh(script: 'curl -o /dev/null -s -w "%{http_code}" http://demo-app-${props.targetEnv}.blue-green.svc.cluster.local:5000/version', returnStdout: true).trim()
                    echo "test response status code: ${statusCode}"
                    if (statusCode == '200') {
                        props.testing = "passed"
                    } else {
                        echo "test run failed: ${statusCode}"
                    }
                }
            }
        }
        stage('Switch Router') {
            when {
                expression { props.testing == "passed" && props.switchEnvs == "true" }
            }
            steps {
                sh 'echo switching'
                sh 'helm upgrade blue-green-ingress --namespace ${props.namespace} --set target=demo-app-${props.targetEnv}'
                sh 'echo "switched app environments, now ${props.targetEnv} is active"'
            }
        }
        stage('live environment test') {
            steps {
                script {
                    def statusCode = sh(script: 'curl -o /dev/null -s -w "%{http_code}" ${props.testEndpoint}', returnStdout: true).trim()
                    echo "live environment test response status code: ${statusCode}"
                    if (statusCode == '200') {
                        props.testing = "passed"
                        echo "live environment test passed"
                    } else {
                        echo "test run failed: ${statusCode}"
                    }
                }
            }
        }
        stage('rollback') {
            when {
                expression { props.testing != "passed" && props.switchEnvs == "true" }
            }
            steps {
                sh 'echo switching back due to failed test'
                sh 'helm upgrade blue-green-ingress --namespace ${props.namespace} --set target=demo-app-${props.otherEnv}'
                sh 'echo "switched back app environments, now ${props.otherEnv} is active"'
            }
        }
    }
}
