pipeline {
    agent any

    options {
        timeout(time: 10, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
    }

    environment {
        APP_NAME  = 'calculator'
        SRC_DIR   = 'src'
        TESTS_DIR = 'tests'
        PYTHON    = 'python3'
        NOTIFY_EMAIL = credentials('notification-email')
    }

    parameters {
        choice(
            name: 'TARGET_ENV',
            choices: ['development', 'staging', 'production'],
            description: 'Target environment for this build'
        )
        booleanParam(
            name: 'RUN_TESTS',
            defaultValue: true,
            description: 'Whether to run unit tests'
        )
        booleanParam(
            name: 'DRY_RUN',
            defaultValue: true,
            description: 'Simulate deployment without executing'
        )
    }

    stages {
        stage('Checkout') {
            steps {
                echo "Build #${env.BUILD_NUMBER} started"
                echo "App        : ${env.APP_NAME}"
                echo "Branch     : ${env.GIT_BRANCH}"
                echo "Commit     : ${env.GIT_COMMIT}"
                echo "Environment: ${params.TARGET_ENV}"
            }
        }

        stage('Build') {
            steps {
                echo "Verifying Python environment..."
                sh 'python3 --version'
                sh 'pytest --version'
                sh '''
                    echo "Source files:"
                    find ${SRC_DIR} -name "*.py"
                '''
            }
        }

        stage('Test') {
            when {
                expression { return params.RUN_TESTS == true }
            }
            steps {
                echo "Running unit tests..."
                sh 'python3 -m pytest ${TESTS_DIR}/ -v'
            }
            post {
                success {
                    echo "✅ All tests passed"
                }
                failure {
                    echo "❌ Tests failed — review pytest output above"
                    echo "Commit that broke tests: ${env.GIT_COMMIT}"
                }
            }
        }

        stage('Deploy') {
            environment {
                DEPLOY_TIMESTAMP = "${new Date().format('yyyy-MM-dd HH:mm:ss')}"
            }
            steps {
                script {
                    if (params.DRY_RUN) {
                        echo "DRY RUN — simulating deployment to ${params.TARGET_ENV}"
                        echo "Would have deployed at: ${env.DEPLOY_TIMESTAMP}"
                    } else {
                        echo "DEPLOYING ${env.APP_NAME} to ${params.TARGET_ENV}"
                        echo "Deployment timestamp: ${env.DEPLOY_TIMESTAMP}"
                        if (params.TARGET_ENV == 'production') {
                            echo "*** PRODUCTION DEPLOYMENT — Build #${env.BUILD_NUMBER} ***"
                        }
                    }
                }
            }
        }

        stage('Summary') {
            steps {
                echo "App        : ${env.APP_NAME}"
                echo "Branch     : ${env.GIT_BRANCH}"
                echo "Build #    : ${env.BUILD_NUMBER}"
                echo "Environment: ${params.TARGET_ENV}"
                echo "Tests ran  : ${params.RUN_TESTS}"
                echo "Dry run    : ${params.DRY_RUN}"
            }
        }
    }

post {
    always {
        echo "Build #${env.BUILD_NUMBER} finished — Status: ${currentBuild.currentResult}"
        cleanWs()
    }
    success {
        echo "✅ ${env.APP_NAME} build #${env.BUILD_NUMBER} succeeded"
        mail(
            to: "${env.NOTIFY_EMAIL}",
            subject: "✅ Jenkins — ${env.APP_NAME} Build #${env.BUILD_NUMBER} Succeeded",
            body: """
                Build succeeded on ${params.TARGET_ENV}.

                Job      : ${env.JOB_NAME}
                Build #  : ${env.BUILD_NUMBER}
                Branch   : ${env.GIT_BRANCH}
                Commit   : ${env.GIT_COMMIT}
                URL      : ${env.BUILD_URL}
                            """
                        )
                    }
    failure {
        echo "❌ ${env.APP_NAME} build #${env.BUILD_NUMBER} failed"
        mail(
            to: "${env.NOTIFY_EMAIL}",
            subject: "❌ Jenkins — ${env.APP_NAME} Build #${env.BUILD_NUMBER} FAILED",
            body: """
                Build failed on ${params.TARGET_ENV}.

                Job      : ${env.JOB_NAME}
                Build #  : ${env.BUILD_NUMBER}
                Branch   : ${env.GIT_BRANCH}
                Commit   : ${env.GIT_COMMIT}
                URL      : ${env.BUILD_URL}

                Review the console output at the URL above.
            """
        )
    }
    fixed {
        echo "🔧 Build is back to green"
    }
    changed {
        echo "⚠️ Build outcome changed from previous run"
    }
}
}