def BASE_URL
def STACK
pipeline {
    agent none

    stages {
        stage('Get Code') {
            agent {label 'aws'}
            steps {
                withCredentials([gitUsernamePassword(credentialsId: 'githubTokenCP1.4')]){
                    script{
                        STACK = "staging"
                            try {
                                sh "git clone https://github.com/ajospino/todo-list-aws-config.git && cd todo-list-aws-config && git checkout $STACK"
                            } catch (err) {
                                echo "No se clonó el repo por el error de arriba"
                            }
                    }
                }
            }
        }

        stage('Deploy'){
            agent {label 'aws'}
            steps{
                script {
                    try {
                        sh """
                            sam build
                            sam deploy --config-file todo-list-aws-config/samconfig.toml --resolve-s3 --config-env $STACK
                        """
                    } catch (err){
                        echo "No se ejecuto completamente el comando del SAM por el error de arriba"
                    }


                    BASE_URL = sh ( script: "aws cloudformation describe-stacks --stack-name todo-list-aws-$STACK --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' --region us-east-1 --output text"
                    , returnStdout: true
                    ).trim()
                }
            }
        }

        stage('Tests')
        {          
            parallel {
                stage('Static') { 
                    agent {label 'nux'}
                    steps {
                        sh 'python -m flake8 --exit-zero --output-file=result-static.xml src/'
                        recordIssues tools: [flake8(pattern: 'result-static.xml')] 
                    }    
                }

                stage('Security') {
                    agent {label 'nux'}
                    steps {
                        sh 'python -m bandit -r src/ -f xml -o result-security.xml'
                        recordIssues tools: [pyLint(name:'Security', pattern: 'result-security.xml')]                        
                    }    
                }

                stage('Rest') {
                    agent {label 'nux'}
                    steps {
                        script{
                            try{
                                sh """
                                    export BASE_URL=$BASE_URL
                                    python -m pytest --junitxml=result-rest.xml test/integration
                                """
                            } catch (err){
                                echo "No se ejecutaron las pruebas por el error de arriba"
                                throw err
                            }
                        }
                    }    
                } 
            } 
        }
        
        stage('Promote'){
            agent {label 'nux'}
            steps{
                sh '''
                    git checkout develop
                    git checkout master
                    git config pull.rebase false
                    git config --global merge.ours.driver true
                    git pull origin master
                    git merge develop
                    git push
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