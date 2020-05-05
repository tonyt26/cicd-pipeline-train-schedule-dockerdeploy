def cancelPreviousBuilds(currentCommitID, prevCommitID, jobName) {
    //def jobName = env.JOB_NAME
    def currentBranch = env.BRANCH_NAME
    def currentBuildNumber = env.BUILD_NUMBER.toInteger()
    def currentJob = Jenkins.instance.getItemByFullName(jobName)
    
    //echo $currentBranch
    echo "Current Job Name: ${currentJob}"
    echo "Current Branch Name: ${currentBranch}"
    echo "Current Commit ID: ${currentCommitID}"
    echo "Previous Commit ID: ${prevCommitID}"
    for (def build : currentJob.builds) {
        if (build.isBuilding() && (build.number.toInteger() < currentBuildNumber) && (currentCommitID != previousCommitID)) {
        echo "Older build still queued. Sending kill signal to build number: ${build.number}"
        build.doStop()
        }
    }
}

pipeline {
    agent any
    stages {
        stage('Kill old builds - Branch 2') {
            parameters {
              string(name: 'currentCommitID', description: 'Current Commit ID', defaultValue: "${env.GIT_COMMIT}")
              string(name: 'prevCommitID', description: 'Previous Commit ID', defaultValue: "${env.GIT_PREVIOUS_COMMIT}")
              string(name: 'jobName', description: 'jobName', defaultValue: "${env.JOB_NAME}")
            }
            steps {
                echo currentCommitID
                echo prevCommitID
                cancelPreviousBuilds(currentCommitID, prevCommitID, jobName)
            }
        }
        stage('Build') {
            steps {
                echo 'Running build automation'
                sh './gradlew build --no-daemon'
                sh 'sleep 100'
                archiveArtifacts artifacts: 'dist/trainSchedule.zip'
            }
        }
        stage('Build Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    app = docker.build("willbla/train-schedule")
                    app.inside {
                        sh 'echo $(curl localhost:8080)'
                    }
                }
            }
        }
        stage('Push Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker_hub_login') {
                        app.push("${env.BUILD_NUMBER}")
                        app.push("latest")
                    }
                }
            }
        }
        stage('DeployToProduction') {
            when {
                branch 'master'
            }
            steps {
                input 'Deploy to Production?'
                milestone(1)
                withCredentials([usernamePassword(credentialsId: 'webserver_login', usernameVariable: 'USERNAME', passwordVariable: 'USERPASS')]) {
                    script {
                        sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$prod_ip \"docker pull ating3/train-schedule:${env.BUILD_NUMBER}\""
                        try {
                            sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$prod_ip \"docker stop train-schedule\""
                            sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$prod_ip \"docker rm train-schedule\""
                        } catch (err) {
                            echo: 'caught error: $err'
                        }
                        sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$prod_ip \"docker run --restart always --name train-schedule -p 8080:8080 -d willbla/train-schedule:${env.BUILD_NUMBER}\""
                    }
                }
            }
        }
    }
}
