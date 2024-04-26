pipeline {
    agent {
        label '4gb-vm-agent'
    } 

    environment {
        CI = 'true'
        VERSION = extractVersionNumber()
        REPO_NAME = extractRepositoryName()
        NODE_IMAGE = 'node:16'
        ARTIFACTORY_URL='demosiemensali.jfrog.io'
        ARTIFACTORY_USERNAME=credentials('JFrogUsername')
        ARTIFACTORY_PASSWORD=credentials('JFrogPassword')
        HOST_USERNAME='ubuntu'
        HOST_IP='13.49.44.93'
    }

    stages {
        stage('Generate Release Tag') {
            steps {
                echo 'Generating release tag...'
                echo "RELEASE TAG: $VERSION"
                sh "git tag $VERSION"
                sh "git push origin --tags"
            }
        }
        
        stage('Install dependencies') {
            agent {
                docker {
                    image "$NODE_IMAGE"
                    reuseNode true
                }
            }
            steps{
            script {
            if (hasFileChanged("package.json") || !fileExists('node_modules')) {
                        echo "package.json has changed. Installing dependencies..."
                        sh 'yarn install'
            } else {
                        echo "File package.json has not changed. Skipping dependency installation."
                    }
            }
            }
        }
        
        stage('Generate Build') {
            agent {
                docker {
                    image "$NODE_IMAGE"
                    reuseNode true
                }
            }
            steps {
                echo 'Generating build folder...'
                sh 'yarn build'
            }
        }

        stage('Run Tests') {
            agent {
                docker {
                    image "$NODE_IMAGE"
                    reuseNode true
                }
            }
            steps {
                 echo 'Running Tests...'
                 sh 'yarn test'
            }
        }
        
        stage('Build & Push Docker Image') {
            steps {
                script {
                    sh "docker login -u \$ARTIFACTORY_USERNAME -p \$ARTIFACTORY_PASSWORD $ARTIFACTORY_URL"
                    echo "Generating $ARTIFACTORY_URL/$REPO_NAME/$REPO_NAME:$VERSION image..."
                    sh "docker build -t $ARTIFACTORY_URL/$REPO_NAME/$REPO_NAME:$VERSION ."
                    sh "docker push $ARTIFACTORY_URL/$REPO_NAME/$REPO_NAME:$VERSION"
                }
            }
        }
        stage('Deploy Docker Image') {
            steps {
                script {
                    withCredentials(bindings: [sshUserPrivateKey(credentialsId: 'HostKey', keyFileVariable: 'HOST_PRIVATE_KEY')]) {
                    sh """ssh -i $HOST_PRIVATE_KEY -o StrictHostKeyChecking=no $HOST_USERNAME@$HOST_IP  << EOF
                    docker pull $ARTIFACTORY_URL/$REPO_NAME/$REPO_NAME:$VERSION
                    docker stop $REPO_NAME || true
                    docker run -d -p 80:80 --name $REPO_NAME $ARTIFACTORY_URL/$REPO_NAME/$REPO_NAME:$VERSION
                    exit
                    EOF"""
                }
                }
            }
    }
}
post {
    success {
        script {
            def committerEmail = sh(script: 'git log --format="%ae" | head -1', returnStdout: true).trim()
            echo "Subject: Jenkins Build Success: ${currentBuild.fullDisplayName}"
            echo "Body: The Jenkins build ${currentBuild.fullDisplayName} succeeded. Build URL: ${BUILD_URL}"
            echo "Recipient: ${committerEmail}"
            emailext (
                subject: "Jenkins Build Success: ${currentBuild.fullDisplayName}",
                body: "The Jenkins build ${currentBuild.fullDisplayName} succeeded. Build URL: ${BUILD_URL}",
                to: "${committerEmail}",
            )
        }
    }
    failure {
        script {
            def committerEmail = sh(script: 'git log --format="%ae" | head -1', returnStdout: true).trim()
            echo "Subject: Jenkins Build Failure: ${currentBuild.fullDisplayName}"
            echo "Body: The Jenkins build ${currentBuild.fullDisplayName} failed. Build URL: ${BUILD_URL}"
            echo "Recipient: ${committerEmail}"
            emailext (
                subject: "Jenkins Build Failed: ${currentBuild.fullDisplayName}",
                body: "The Jenkins build ${currentBuild.fullDisplayName} failed. Build URL: ${BUILD_URL}",
                to: "${committerEmail}",
            )
        }
    }
}

}



def extractRepositoryName() {
    return sh(script: 'basename -s .git ${GIT_URL}', returnStdout: true).trim()
}
def extractVersionNumber(){
    def packageJson = readJSON file: 'package.json'
    return packageJson.version 
}
def hasFileChanged(file) {
    def currentCommit = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
    def previousCommit = sh(script: 'git rev-parse HEAD^', returnStdout: true).trim()

    def diffCommand = "git diff --name-only ${previousCommit} ${currentCommit}"
    def diffOutput = sh(script: diffCommand, returnStdout: true).trim()

    return diffOutput.contains(file)
}