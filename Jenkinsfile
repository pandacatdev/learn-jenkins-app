pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = '9804ee5a-c8ff-40f7-aa94-11bb7fecbaba'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
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
                    npm --version
                    node --version
                    ls -la
                    echo "Installing dependencies..."
                    npm ci
                    echo "Building the project..."
                    npm run build
                    ls -la
                '''
            }
        }

        stage('Verify') {
            parallel {
                stage('Unit Test') {
                    agent {
                        docker {
                            image 'node:18-alpine'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            echo "Running tests..."
                            test -f build/index.html || (echo "Build failed, skipping tests" && exit 1)
                            npm test
                            echo "Tests completed successfully"
                        '''
                    }
                    post {
                        always {
                            // Change default path to jest-results to avoid conflicts between junit and playwright
                            junit 'jest-results/junit.xml'
                        }
                    }
                }

                stage('E2E Test') {
                    agent {
                        docker {
                            image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            echo "Running E2E tests..."

                            # Install serve, not install globaly to avoid permission issues
                            npm install serve
                            
                            # Use serve in node_modules/.bin
                            # '&' to run in background and allow the next command to run
                            node_modules/.bin/serve -s build & 

                            # Wait for 10 seconds to ensure the server is up
                            sleep 10

                            # Run Playwright tests
                            npx playwright test --reporter=html
                        '''
                    }
                    post {
                        always {
                            // Publish Playwright HTML report
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright E2E Local', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }
            }
        }

        stage('Deploy Staging') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    echo "Deploying the project to staging..."
                    npm install netlify-cli
                    node_modules/.bin/netlify --version
                    node_modules/.bin/netlify status

                    echo "Deploy without --prod flag to create a draft deploy"
                    node_modules/.bin/netlify deploy --dir build
                '''
            }
        }

        stage('Deploy Prod') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                timeout(time: 1, unit: 'MINUTES') {
                    input message: 'Are you sure to deploy?', ok: 'Deploy'
                }
                sh '''
                    echo "Deploying the project..."
                    npm install netlify-cli
                    node_modules/.bin/netlify --version
                    echo "Deploying to Netlify Site ID: $NETLIFY_SITE_ID..."
                    node_modules/.bin/netlify status
                    node_modules/.bin/netlify deploy --dir build --prod
                    echo "Deployment completed successfully"
                '''
            }
        }

        stage('E2E Prod') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }

            environment {
               CI_ENVIRONMENT_URL = "https://beamish-kitsune-b93ae9.netlify.app"
            }

            steps {
                sh '''
                    echo "Running E2E tests on Prod..."
                    npx playwright test --reporter=html
                '''
            }
            post {
                always {
                    // Publish Playwright HTML report
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright E2E Prod', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }
    }
}