pipeline {
    agent any

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
        stage('Test') {
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
        }
    }

    post {
        always {
            junit 'test-results/junit.xml'
        }
    }
}