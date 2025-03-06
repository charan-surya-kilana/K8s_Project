pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'nodejs'
    }

    environment {
        SCANNER_HOME = tool 'mysonar'
    }

    stages {
        stage ("Clean Workspace") {
            steps {
                cleanWs()
            }
        }

        stage ("Clone Repository") {
            steps {
                git branch: 'main', url: 'https://github.com/devops0014/Zomato-Repo.git'
            }
        }

        stage("SonarQube Analysis") {
            steps {
                withSonarQubeEnv('mysonar') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=zomato \
                        -Dsonar.projectKey=zomato
                    '''
                }
            }
        }

        stage ("Quality Gates") {
            steps {
                script {
                    waitForQualityGate abortPipeline: true, credentialsId: 'sonar_password'
                }
            }
        }

        stage ("Install Dependencies") {
            steps {
                sh 'npm install'
            }
        }

        stage ("Trivy File System Scan") 
        {
            steps 
            {
                sh "trivy fs . > trivyfs.txt"
            }
        }

        stage ("Build Docker Image") {
            steps {
                sh 'docker build -t image1 .'
            }
        }

        stage ("Docker Build & Push") {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub') {
                        sh '''
                            docker tag image1 charansuryakilana/ez_deploy_prototype:image1
                            docker push charansuryakilana/ez_deploy_prototype:image1
                        '''
                    }
                }
            }
        }

        stage ("Scan Docker Image") {
            steps {
                sh 'trivy image charansuryakilana/ez_deploy_prototype:image1'
            }
        }
    }
    post {
    always {
        echo 'Slack Notifications'
        slackSend (
            channel: 'ezdeploy-prototype',
            message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} \n Build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
        )
    }
}

}
