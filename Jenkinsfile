def mainDir=""
def region="ap-northeast-2"
def nexusUrl="ec2-15-164-164-214.ap-northeast-2.compute.amazonaws.com:5000"
def repository="test"
def deployHost="172.31.33.197"
def tagName="nexus"
def nexusid="admin"
def nexuspw="admin"
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
                cd ${mainDir}
                ./gradlew clean build
                """
            }
        }
        stage('Build Docker Image by Jib & Push to Nexus Registry') {
            steps {
                sh """
                    cd ${mainDir}
                    docker login -u ${nexusid} -p ${nexuspw} ${nexusUrl}
                    ./gradlew jib -Djib.to.image=${nexusUrl}/${repository}:${tagName} -DsendCredentialsOverHttp=true -Djib.console='plain'
                """
            }
        }
        stage('Scan Security CVE at Clair Scanner') {
            steps {
                script {
                    try {
                        clair_ip = sh(script: "docker inspect -f '{{ .NetworkSettings.IPAddress }}' clair", returnStdout: true).trim()
                        sh """
                            docker pull ${nexusUrl}/${repository}:${tagName}
                            clair-scanner --ip ${jenkins_ip} --clair='http://${clair_ip}:6060' --log='clair.log' --report='report.txt' ${nexusUrl}/${repository}:${tagName}
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
                     'docker login -u ${nexusid} -p ${nexuspw} ${nexusUrl}; \
                      docker run -d -p 80:8080 -t ${nexusUrl}/${repository}:${tagName};'"
                }
            }
        }
    }
}