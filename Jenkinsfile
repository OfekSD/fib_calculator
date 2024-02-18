def app
pipeline {
    agent {
        kubernetes {
            defaultContainer 'docker'
            yaml '''apiVersion: v1
kind: Pod
spec:
  volumes:
    - name: dockersock
      hostPath:
        path: /var/run/docker.sock
    - name: helm
      hostPath:
        path: /usr/local/bin/helm
  containers:
  - name: docker
    image: docker
    command:
    - sleep
    args:
    - infinity
    volumeMounts:
    - name: dockersock
      mountPath: "/var/run/docker.sock"
  - name: helm
    image: alpine/helm
    command:
    - sleep
    args:
    - infinity
    volumeMounts:
    - name: helm
      mountPath: "/usr/local/bin/helm"
  - name: git
    image: bitnami/git
    command:
    - sleep
    args:
    - infinity
  - name: node
    image: node:16-alpine
    command:
    - sleep
    args:
    - infinity
  - name: yq
    image: linuxserver/yq
    command:
    - sleep
    args:
    - infinity
  '''
        }
    }

    stages {
        stage('checkout') {
            steps {
                git branch: 'main', credentialsId: 'github-creds', url: 'git@github.com:OfekSD/fib_calculator_k8s.git'
            }
        }

        stage('server - Test Code') {
                when { changeset 'server/**' }
                steps {
                    container('node') {
                    dir('server') {
                        sh 'npm install'
                        sh 'npm test'
                    }
                    }
                }
        }
        stage('server - Build Image') {
                when { changeset 'server/**' }
                steps {
                    dir('server') {
                        script {
                            app = docker.build('pandalamdta/server')
                        }
                    }
                }
        }
        stage('server - Push image') {
            when { changeset 'server/**' }
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-credentials') {
                        app.push("${env.BUILD_NUMBER}")
                        app.push('latest')
                    }
                }
            }
        }
        stage('server - Update Helm Chart Image') {
            when { changeset 'server/**' }
            steps {
                    container('yq') {
                        sh "yq -i .server.image.tag =  ${env.BUILD_NUMBER}"
                    }
            }
        }
        stage('worker - Test Code') {
            when { changeset 'worker/**' }
            steps {
                container('node') {
                    dir('worker') {
                        sh 'npm install'
                        sh 'npm test'
                    }
                }
            }
        }
        stage('worker - Build Image') {
                when { changeset 'worker/**' }
                steps {
                    dir('worker') {
                        script {
                            app = docker.build('pandalamdta/worker')
                        }
                    }
                }
        }
        stage('worker - Push image') {
                when { changeset 'worker/**' }
                steps {
                        script {
                        docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-credentials') {
                            app.push("${env.BUILD_NUMBER}")
                            app.push('latest')
                        }
                        }
                }
        }
        stage('worker - Update Helm Chart Image') {
                when { changeset 'worker/**' }
                steps {
                        container('yq') {
                            sh "yq -i .worker.image.tag =  ${env.BUILD_NUMBER}"
                        }
                }
        }

        stage('client - Test Code') {
            when { changeset 'client/**' }
            steps {
                container('node') {
                    dir('client') {
                        sh 'npm install'
                        sh 'npm test'
                    }
                }
            }
        }
        stage('client - Build Image') {
                when { changeset 'client/**' }
                steps {
                    dir('client') {
                        script {
                            app = docker.build('pandalamdta/client')
                        }
                    }
                }
        }
        stage('client - Push image') {
                when { changeset 'client/**' }
                steps {
                        script {
                        docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-credentials') {
                            app.push("${env.BUILD_NUMBER}")
                            app.push('latest')
                        }
                        }
                }
        }
        stage('client - Update Helm Chart Image') {
            when { changeset 'client/**' }
            steps {
                container('yq') {
                    sh "yq -i .client.image.tag =  ${env.BUILD_NUMBER}"
                }
            }
        }
        stage('Helm - Lint') {
            steps {
                container('helm') {
                    sh 'helm lint ./helm'
                }
            }
        }
        stage('Helm - Test') {
            steps {
                script {
                    container('git') {
                        sh "git config --global --add safe.directory '*'"
                        sh 'git status'
                        
                        def changesStatus = sh script: 'git status | grep modified | grep helm', returnStatus: true
                    }
                    if (changesStatus == 0) {
                        container('helm') {
                            sh 'helm install fib-test ./helm  --create-namespace'
                            def testStatus = sh script: 'helm test fib-test', returnStatus: true
                            if (testStatus > 0) {
                                sh 'helm uninstall fib-test '
                                error: 'helm test failed'
                            }
                            sh 'helm uninstall fib-test '
                        }
                    }
                }
            }
        }
        stage('Git Push Changes') {
            steps {
                container('git') {
                    script {
                        def changesStatus = sh script: 'git status | grep modified | grep helm', returnStatus: true
                        if (changeStatus == 0) {
                            withCredentials([usernamePassword(credentialsId: 'github-creds', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                                sh "git config --global --add safe.directory '*'"
                                sh 'git config user.email jenkins@example.com'
                                sh 'git config user.name jenkins-pipeline'
                                sh 'git add ./helm'
                                sh "git commit -m 'Triggered Build: ${env.BUILD_NUMBER}' updated helm"
                                sh 'git push'
                            }
                        }
                    }
                }
            }
        }
    }
}

