pipeline {
    
    agent any

    environment {
        NETLIFY_SITE_ID = '537a6fbd-5c67-4360-bda1-d46c91c80257'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
        REACT_APP_VERSION = "1.0.$BUILD_ID"
        DEPLOYMENT_TOKEN = credentials('azure-deploy-token')
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
        stage('Azure Login') {
            agent {
                docker {
                image 'mcr.microsoft.com/azure-cli'
                args '-u root'
                }
            }
            steps {
                withCredentials([azureServicePrincipal(
                credentialsId: 'jenkins-static-swa-sp',
                clientIdVariable: 'AZURE_CLIENT_ID',
                clientSecretVariable: 'AZURE_CLIENT_SECRET',
                tenantIdVariable: 'AZURE_TENANT_ID',
                subscriptionIdVariable: 'AZURE_SUBSCRIPTION_ID'
                )]) {
                sh '''
                    echo "Installing SWA CLI via npm"
                    npm install -g @azure/static-web-apps-cli || true

                    echo "Azure Login"
                    az login --service-principal \
                    -u $AZURE_CLIENT_ID \
                    -p $AZURE_CLIENT_SECRET \
                    -t $AZURE_TENANT_ID

                    az account set --subscription $AZURE_SUBSCRIPTION_ID
                '''
                }
            }
        }
        stage('Install and Deploy SWA') {
            agent {
                docker {
                image 'node:18'
                args '-u root'
                }
            }
            steps {
                sh '''
                mkdir -p ~/.npm-global
                npm config set prefix '~/.npm-global'
                export PATH=~/.npm-global/bin:$PATH
                npm install -g @azure/static-web-apps-cli

                echo "Deploying to Azure Static Web App"
                swa deploy --app-location src --env production --deployment-token $DEPLOYMENT_TOKEN
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