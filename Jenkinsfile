pipeline{
    agent any
    tools{
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    parameters {
        string(name: 'DOCKER_IMAGE_TAG', defaultValue: '${BUILD_NUMBER}', description: 'Enter the Docker image tag')
        string(name: 'GITHUB_REPO_URL', defaultValue: 'https://github.com/IVISHSAGGAM/Netflix-K8s.git', description: 'URL of the GitHub repository containing Kubernetes manifest files')
        string(name: 'GITHUB_CREDENTIALS_ID', defaultValue: 'GitHub', description: 'Credentials ID for accessing the GitHub repository (if needed)')
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git branch: 'main', url: 'https://github.com/IVISHSAGGAM/Netflix-Clone.git'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix '''
                }
            }
        }
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' 
                }
            } 
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){   
                       sh "docker build --build-arg TMDB_V3_API_KEY=16e409c969fcb38173f6b16f05725a8f -t netflix ."
                       sh "docker tag netflix ivishsaggam/netflix:${BUILD_NUMBER} "
                       sh "docker push ivishsaggam/netflix:${BUILD_NUMBER} "
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image ivishsaggam/netflix:${BUILD_NUMBER} > trivyimage.txt" 
            }
        }
        stage('Clone Repository') {
            steps {
                script {
                    // Clone the GitHub repository
                    git branch: 'main', credentialsId: "${params.GITHUB_CREDENTIALS_ID}", url: "${params.GITHUB_REPO_URL}"
                }
            }
        }
        
        stage('Find YAML files') {
            steps {
                script {
                    // Directory where Kubernetes manifest files are stored
                    def manifestDir = 'kubernetes'

                    // Print out the files found by the find command
                    sh "find ${manifestDir} -type f -name '*.yaml' -print"
                }
            }
        }

        stage('Update Image Tag') {
            steps {
                script {
                    withCredentials([gitUsernamePassword(credentialsId: 'GitHub', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]){
                    
                    // Directory where Kubernetes manifest files are stored
                    def manifestDir = 'kubernetes'

                    // Docker image tag to be updated
                    def newImageTag = "${params.DOCKER_IMAGE_TAG}"

                    // Iterate through each YAML file in the manifest directory
                    sh """
    find kubernetes -type f -name deployment.yaml -exec sh -c 'sed -i "s/image:.*/image: ivishsaggam\\/netflix:${newImageTag}/g" "{}" && cat "{}"' \\;
""" 
                    sh "cd ${manifestDir} && git add . && git commit -m 'Update Docker image tag' && git remote add netflix https://github.com/IVISHSAGGAM/Netflix-K8s.git && git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/IVISHSAGGAM/Netflix-K8s.git HEAD:main "
                
                    }
                }
                
            }
        }
        


    }
    post {
     always {
        emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: "Project: ${env.JOB_NAME}<br/>" +
                "Build Number: ${env.BUILD_NUMBER}<br/>" +
                "URL: ${env.BUILD_URL}<br/>",
            to: 'vishalsaggam2@gmail.com',
            attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
    
    
}
