pipeline {
    agent any
 
    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'test', 'prod'],
            description: 'Select target Layer7 environment'
        )
    }
 
    environment {
        BUNDLE_FILE = 'bundles/Configuration-Cache-Demo.bundle'
        GMU_HOME = '/opt/gmu'
    }
 
    stages {
 
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
 
        stage('Load Environment Configuration') {
            steps {
                script {
                    def envConfig = readYaml file: "config/${params.ENVIRONMENT}.yaml"
 
                    env.GATEWAY_HOST = envConfig.gateway.host
                    env.GATEWAY_PORT = envConfig.gateway.port.toString()
                    env.GATEWAY_PROTOCOL = envConfig.gateway.protocol
 
                    echo "Target Environment : ${params.ENVIRONMENT}"
                    echo "Gateway Host       : ${env.GATEWAY_HOST}"
                    echo "Gateway Port       : ${env.GATEWAY_PORT}"
                }
            }
        }
 
        stage('Validate Bundle') {
            steps {
                script {
                    if (!fileExists(env.BUNDLE_FILE)) {
                        error "Bundle file not found: ${env.BUNDLE_FILE}"
                    }
 
                    echo "Bundle found: ${env.BUNDLE_FILE}"
                }
            }
        }
 
        stage('Validate GMU') {
            steps {
                sh '''
                    echo "Checking GMU installation..."
 
                    if [ ! -f "${GMU_HOME}/GatewayMigrationUtility.sh" ]; then
                        echo "GMU not found at ${GMU_HOME}"
                        exit 1
                    fi
 
                    echo "GMU installation found."
                '''
            }
        }
 
        stage('Deploy to Layer7') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'layer7-gateway-credentials',
                        usernameVariable: 'GATEWAY_USERNAME',
                        passwordVariable: 'GATEWAY_PASSWORD'
                    )
                ]) {
 
                    sh '''
                        echo "Deploying bundle to Layer7 Gateway..."
 
                        ${GMU_HOME}/GatewayMigrationUtility.sh \
                            migrateIn \
                            --host ${GATEWAY_HOST} \
                            --port ${GATEWAY_PORT} \
                            --username "${GATEWAY_USERNAME}" \
                            --password "${GATEWAY_PASSWORD}" \
                            --bundle "${BUNDLE_FILE}"
 
                        echo "Deployment completed."
                    '''
                }
            }
        }
 
        stage('Deployment Verification') {
            steps {
                echo "GMU deployment completed successfully."
            }
        }
    }
 
    post {
 
        success {
            echo """
            ==========================================
            Layer7 Deployment SUCCESS
            Environment : ${params.ENVIRONMENT}
            Bundle      : ${env.BUNDLE_FILE}
            ==========================================
            """
        }
 
        failure {
            echo """
            ==========================================
            Layer7 Deployment FAILED
            Environment : ${params.ENVIRONMENT}
            ==========================================
            """
        }
 
        always {
            cleanWs()
        }
    }
}