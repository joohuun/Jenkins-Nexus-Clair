def mainDir=""
def region="ap-northeast-2"
def nexusUrl="ip-172-31-33-247.ap-northeast-2.compute.internal:5443"
def repository="container-registry"
def deployHost="172.31.33.197"
def tagName="nexus"
def nexusid="test"
def nexuspw="test"
def jenkins_ip="172.31.46.116"

pipeline {
    agent any

    stages {
        stage('Pull Codes from Github'){
            steps{
                checkout scm
            }
        }
        stage('Build Codes by Gradle') {
            steps {
                sh """
                ./gradlew clean build
                """
            }
        }
        stage('Build Docker Image by Jib & Push to Nexus Registry') {
            steps {
                sh """
                    docker login -u ${nexusid} -p ${nexuspw} ${nexusUrl}
                    ./gradlew jib -Djib.to.image=${nexusUrl}/${repository}:${tagName} -Djib.console='plain'
                """
            }
        }
        stage('Scan Security CVE at Clair Scanner') {
            steps {
                script {
                    try {
                        clair_ip = sh(script: "docker inspect -f '{{ .NetworkSettings.IPAddress }}' clair", returnStdout: true).trim()
                        sh """
                            apt udate
                            apt install -y wget
                            docker pull ${nexusUrl}/${repository}:${tagName}
                            wget https://github.com/arminc/clair-scanner/releases/download/v12/clair-scanner_linux_amd64
                            chmod +x clair-scanner_linux_amd64;
                            sudo mv clair-scanner_linux_amd64 /usr/local/bin/clair-scanner
                            clair-scanner --ip 172.17.0.1 --clair='http://${clair_ip}:6060' --log='clair.log' --report='report.txt' ${nexusUrl}/${repository}:${tagName}
                        """
                    } catch (err) {
                        echo err.getMessage()
                    }
                }
                echo currentBuild.result
            }
        }
        stage('Deploy container to AWS EC2 VM'){
            steps{
                sshagent(credentials : ["deploy-key"]) {
                    sh "ssh -o StrictHostKeyChecking=no ubuntu@${deployHost} \
                     'sudo docker login -u ${nexusid} -p ${nexuspw} ${nexusUrl}; \
                      sudo docker run -d -p 80:8080 -t ${nexusUrl}/${repository}:${tagName};'"
                }
            }
        }
    }
}
