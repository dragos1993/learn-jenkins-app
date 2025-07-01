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
        stage('Deploy in Azure') {
            agent {
                docker {
                    image 'mcr.microsoft.com/azure-cli'
                    reuseNode true
                }
            }
            environment {
                AZURE_CLIENT_ID     = credentials('azure-client-id')
                AZURE_CLIENT_SECRET = credentials('azure-client-secret')
                AZURE_TENANT_ID     = credentials('azure-tenant-id')
                AZURE_STORAGE_ACCOUNT = 'jenkinsapp' // Replace with your Azure Storage account
                AZURE_CONFIG_DIR = './.azure' //
            }
            steps {
                sh '''
                    echo "Logging into Azure..."
                    mkdir -p .azure
                    echo "Using tenant ID: $AZURE_TENANT_ID"
                    echo "Using secret ID: $AZURE_CLIENT_SECRET"
                    echo "Using client ID: $AZURE_CLIENT_ID"
                    AZURE_CONFIG_DIR=./.azure az login --service-principal \\
                    -u $AZURE_CLIENT_ID \\
                    -p $AZURE_CLIENT_SECRET \\
                    --tenant $AZURE_TENANT_ID

                    echo "Uploading build folder to Azure Blob..."
                    az storage blob upload-batch \\
                    --account-name $AZURE_STORAGE_ACCOUNT \\
                    --destination \$web \\
                    --source build \\
                    --overwrite
                '''
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