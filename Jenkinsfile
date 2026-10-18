// Sanitized example pipeline: GitLab -> Jenkins -> deploy to private-subnet servers via bastion.
// All names, hosts, credentials IDs and URLs below are dummy values.

pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 30, unit: 'MINUTES')
    }

    parameters {
        choice(name: 'TARGET_ENV', choices: ['mumbai-prod', 'singapore-prod'], description: 'Environment to deploy to')
        string(name: 'SERVICE_NAME', defaultValue: 'sample-service', description: 'Service to deploy')
    }

    environment {
        APP_NAME     = "${params.SERVICE_NAME}"
        ARTIFACT_DIR = 'dist'
        BASTION_HOST = 'bastion.example.internal'
        APP_HOSTS    = '10.0.1.10,10.0.1.11'          // private subnet servers (dummy IPs)
        DEPLOY_USER  = 'deploy'
        HEALTH_URL   = 'https://api.example.com/health'
        SSH_CRED_ID  = 'deploy-ssh-key'                // Jenkins credentials ID (dummy)
    }

    triggers {
        gitlab(triggerOnPush: true, triggerOnMergeRequest: false, branchFilterType: 'NameBasedFilter', includeBranchesSpec: 'main')
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                sh 'git log -1 --pretty=format:"%h %s"'
            }
        }

        stage('Build') {
            steps {
                sh '''
                    set -e
                    mkdir -p ${ARTIFACT_DIR}
                    # Replace with your real build command (mvn, npm, gradle, etc.)
                    ./build.sh
                    tar -czf ${ARTIFACT_DIR}/${APP_NAME}-${BUILD_NUMBER}.tar.gz -C build .
                '''
            }
        }

        stage('Test') {
            steps {
                sh './run-tests.sh'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'reports/**/*.xml'
                }
            }
        }

        stage('Archive') {
            steps {
                archiveArtifacts artifacts: "${ARTIFACT_DIR}/*.tar.gz", fingerprint: true
            }
        }

        stage('Deploy') {
            steps {
                sshagent(credentials: [env.SSH_CRED_ID]) {
                    script {
                        env.APP_HOSTS.split(',').each { host ->
                            sh """
                                set -e
                                # Jump through the bastion host to reach private-subnet servers
                                SSH_OPTS="-o StrictHostKeyChecking=no -o ProxyJump=${DEPLOY_USER}@${BASTION_HOST}"

                                scp \$SSH_OPTS ${ARTIFACT_DIR}/${APP_NAME}-${BUILD_NUMBER}.tar.gz ${DEPLOY_USER}@${host}:/tmp/

                                ssh \$SSH_OPTS ${DEPLOY_USER}@${host} '
                                    set -e
                                    sudo mkdir -p /opt/${APP_NAME}/releases/${BUILD_NUMBER}
                                    sudo tar -xzf /tmp/${APP_NAME}-${BUILD_NUMBER}.tar.gz -C /opt/${APP_NAME}/releases/${BUILD_NUMBER}
                                    sudo ln -sfn /opt/${APP_NAME}/releases/${BUILD_NUMBER} /opt/${APP_NAME}/current
                                    sudo systemctl restart ${APP_NAME}
                                    rm -f /tmp/${APP_NAME}-${BUILD_NUMBER}.tar.gz
                                '
                            """
                        }
                    }
                }
            }
        }

        stage('Health check') {
            steps {
                retry(5) {
                    sleep time: 10, unit: 'SECONDS'
                    sh 'curl -fsS --max-time 10 ${HEALTH_URL}'
                }
            }
        }
    }

    post {
        success {
            echo "Deployed ${APP_NAME} build ${BUILD_NUMBER} to ${params.TARGET_ENV}"
        }
        failure {
            echo "Deployment failed for ${APP_NAME} build ${BUILD_NUMBER}. Check logs in Graylog and the Jenkins console."
            // Optional: add a notification step here (email, Slack, etc.)
        }
        cleanup {
            cleanWs()
        }
    }
}
