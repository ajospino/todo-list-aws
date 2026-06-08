def base_url
pipeline {
    agent none

    stages {
        stage('Get Code') {
            agent {label 'nux'}
            steps {
                // Obtener código del repo
                git 'https://github.com/ajospino/todo-list-aws.git'
                sh 'git clone https://github.com/ajospino/todo-list-aws-config.git && cd todo-list-aws-config && git checkout production'
            }
        }

        stage('Deploy'){
            agent {label 'aws'}
            steps{
                sh '''
                    sam build
                    sam deploy --config-file todo-list-aws-config/samconfig.toml --resolve-s3
                '''
            }
        }


        stage('Rest') {
            agent {label 'nux'}
            steps {
                base_url = sh "aws cloudformation describe-stacks --stack-name todo-list-aws-production --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' --region us-east-1 --output text"
                sh '''
                    export BASE_URL="$base_url"
                    python -m pytest --junitxml=result-rest.xml test/integration
                '''
            }    
        } 
                
        stage('Cleanup'){
            parallel{
                stage('Clear Workspace for Linux Agent 1'){
                    agent {label 'nux'}
                    steps{
                        cleanWs notFailBuild: true 
                    }
                }

                stage('Clear Workspace for Linux Agent 2'){
                    agent {label 'nux'}
                    steps{
                        cleanWs notFailBuild: true 
                    }
                }

                stage('Clear Workspace for AWS Agent'){
                    agent {label 'aws'}
                    steps{
                        cleanWs notFailBuild: true 
                    }
                }
            }
        }
    }
}