pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = '537a6fbd-5c67-4360-bda1-d46c91c80257'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
        REACT_APP_VERSION = "1.0.$BUILD_ID"
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
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    ls -la
                '''
            }
        }

        stage('Tests') {
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
                            echo 'Test stage'
                            test -f build/index.html
                            npm test
                        '''
                    }
                    post {
                        always {
                            junit 'jest-results/junit.xml'
                        }
                    }
                }
            }
        }

        stage('Deploy Azure') {
    environment {
        AZURE_SP_JSON = credentials('jenkins-sp')
    }
    steps {
        script {
            writeFile file: 'azure.json', text: env.AZURE_SP_JSON

            def azure = readJSON file: 'azure.json'

            sh """
            az login --service-principal \\
              -u ${azure.clientId} \\
              -p ${azure.clientSecret} \\
              --tenant ${azure.tenantId}

            az storage blob upload-batch \\
              --account-name jenkinsapp \\
              --destination \$web \\
              --source build \\
              --overwrite
            """
        }
    }
}

        stage('Deploy in Netlify') {
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
                    echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"
                    node_modules/.bin/netlify status
                    node_modules/.bin/netlify deploy --dir=build --prod
                '''
            }
        }
    }
}