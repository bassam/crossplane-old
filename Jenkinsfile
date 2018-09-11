pipeline {
    agent { label 'upbound-gce' }

    options {
        disableConcurrentBuilds()
        timestamps()
    }

    stages {

        stage('Prepare') {
            steps {
                sh 'git config --global user.name "upbound-bot"'
                sh 'echo "machine github.com login upbound-bot password $GITHUB_UPBOUND_BOT" > ~/.netrc'
            }
        }

        stage('Build validation'){
            steps {
                sh './build/run make -j$(nproc) build.all'
            }
        }

        stage('Unit Tests') {
            steps {
                sh './build/run make -j$(nproc) test'
            }
            post {
                always {
                    archive "_output/tests/**/*"
                    junit "_output/tests/**/*.xml"
                }
            }
        }

        stage('Publish') {

            environment {
                DOCKER = credentials('dockerhub-upboundci')
                AWS = credentials('aws-upbound-bot')
                GITHUB_UPBOUND_BOT = credentials('github-upbound-jenkins')
            }

            steps {
                sh 'docker login -u="${DOCKER_USR}" -p="${DOCKER_PSW}"'
                sh "./build/run make -j\$(nproc) -C build/release build BRANCH_NAME=${BRANCH_NAME} GIT_API_TOKEN=${GITHUB_UPBOUND_BOT}"
                sh "./build/run make -j\$(nproc) -C build/release publish BRANCH_NAME=${BRANCH_NAME} AWS_ACCESS_KEY_ID=${AWS_USR} AWS_SECRET_ACCESS_KEY=${AWS_PSW} GIT_API_TOKEN=${GITHUB_UPBOUND_BOT}"
            }
        }

        stage('Promote') {
            when {
                branch 'master'
            }

            steps {
                lock('promote-job') {
                    sh "./build/run make -j\$(nproc) -C build/release promote BRANCH_NAME=master CHANNEL=master AWS_ACCESS_KEY_ID=${AWS_USR} AWS_SECRET_ACCESS_KEY=${AWS_PSW}"
                }
            }
        }

        stage('Deploy') {
            when {
                branch 'master'
            }

            steps {
                build job: 'upbound/cloud/dev/master', parameters: [string(name: 'action', value: 'up')], quietPeriod: 30
            }
        }
    }

    post {
        always {
            script {
                if (params.channel != env.reload) {
                    sh 'make -j\$(nproc) -C build/release clean'
                }
            }
        }
    }
}

