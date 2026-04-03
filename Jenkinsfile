pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = '4e0a3f77-0ae3-4335-967c-a172cd5a0305'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
        REACT_APP_VERSION = "1.0.$BUILD_ID"
        APPSCAN_APP_ID = credentials('appscan-app-id')
    }

    stages {
        stage('Build') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    echo "Building the application..."
                    ls -la
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    ls -la
                '''
            }
        }
        /*
        stage('Lint') {
            agent{
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    npm install eslint
                    npx eslint . --ext .js,.jsx,.ts,.tsx
                '''
            }
        }
        */
        stage('Run Tests Parallel') {
            parallel {
                stage('Unit test') {
                    agent {
                        docker {
                            image 'node:18-alpine'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            test -f build/index.html
                            CI=true npm test -- --watchAll=false --coverage --coverageReporters=lcov
                        '''
                    }
                    post {
                        always {
                            junit 'jest-results/junit.xml'
                        }
                    }
                }

                stage('E2E') {
                    agent {
                        docker {
                            image 'my-playwright'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            serve -s build &
                            sleep 10
                            npx playwright test --reporter=html
                        '''
                    }
                    post {
                        always {
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright Local', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }
            }
        }

        stage('SAST') {
            parallel {
                stage('SonarQube') {
                    steps {
                        withSonarQubeEnv('SonarQube') {
                            sh """
                                ${tool 'sonar-scanner'}/bin/sonar-scanner \
                                    -Dsonar.analysis.buildNumber=${BUILD_NUMBER} \
                                    -Dsonar.links.ci=${BUILD_URL}
                            """
                        }
                    }
                }

                stage('AppScan') {
                    steps {
                        withCredentials([usernamePassword(
                            credentialsId: 'appscan-creds',
                            usernameVariable: 'ASOC_KEY_ID',
                            passwordVariable: 'ASOC_KEY_SECRET'
                        )]) {
                            script {
                                appscan application: "${APPSCAN_APP_ID}",
                                        credentials: 'appscan-creds',
                                        name: 'learn-jenkins-app',
                                        type: 'Static Analyzer',
                                        scanner: staticAnalyzer(target: '.', hasOptions: false),
                                        failBuild: false,
                                        wait: true

                                // AppScan plugin does not support buildUrl on SAST scans.
                                // Post the build URL as a comment via ASoC REST API after scan completes.
                                sh """
                                    TOKEN=\$(curl -s -X POST https://cloud.appscan.com/api/v2/Account/ApiKeyLogin \
                                        -H 'Content-Type: application/json' \
                                        -d '{"KeyId":"'"\$ASOC_KEY_ID"'","KeySecret":"'"\$ASOC_KEY_SECRET"'"}' \
                                        | grep -o '"Token":"[^"]*"' | cut -d'"' -f4)

                                    SCAN_ID=\$(curl -s -X GET "https://cloud.appscan.com/api/v2/Scans?%24filter=Name+eq+'learn-jenkins-app'" \
                                        -H "Authorization: Bearer \$TOKEN" \
                                        | grep -o '"Id":"[^"]*"' | head -1 | cut -d'"' -f4)

                                    curl -s -X POST "https://cloud.appscan.com/api/v2/Scans/\$SCAN_ID/Comments" \
                                        -H "Authorization: Bearer \$TOKEN" \
                                        -H "Content-Type: application/json" \
                                        -d '{"Comment":"Jenkins Build #${BUILD_NUMBER} | ${BUILD_URL}"}'
                                """
                            }
                        }
                    }
                }
            }
        }

        stage('Deploy staging') {
            agent {
                docker {
                    image 'my-playwright'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    netlify --version
                    echo "Deploying to Netlify site ID: $NETLIFY_SITE_ID"
                    netlify status
                    netlify deploy --dir=build --json > deploy-output.json
                '''
                script {
                    env.STAGING_URL = sh(script: "jq -r '.deploy_url' deploy-output.json", returnStdout: true)
                }
            }
        }

        stage('Staging E2E') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }

            environment {
                CI_ENVIRONMENT_URL = "${env.STAGING_URL}"
            }
            steps {
                sh '''
                    npx playwright test --reporter=html
                '''
            }
            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Staging E2E', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }

        stage('Approval') {
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input message: 'Approve deployment to production?', ok: 'Deploy'
                }
            }
        }

        stage('Deploy prod') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    npm install netlify-cli@20.1.1
                    node_modules/.bin/netlify --version
                    echo "Deploying to Netlify site ID: $NETLIFY_SITE_ID"
                    node_modules/.bin/netlify status
                    node_modules/.bin/netlify deploy --dir=build --prod
                '''
            }
        }

        stage('Prod E2E') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }

            environment {
                CI_ENVIRONMENT_URL = 'https://cerulean-pixie-998447.netlify.app/'
            }
            steps {
                sh '''
                    npx playwright test --reporter=html
                '''
            }
            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Prod E2E', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }
    }
}
