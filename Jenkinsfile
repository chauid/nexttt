@Library('my-shared-library') _

void setBuildStatus(String message, String context, String state) {
    step([
        $class: "GitHubCommitStatusSetter",
        reposSource: [$class: "ManuallyEnteredRepositorySource", url: env.GIT_URL],
        contextSource: [$class: "ManuallyEnteredCommitContextSource", context: context],
        errorHandlers: [[$class: "ChangingBuildStatusErrorHandler", result: "UNSTABLE"]],
        statusResultSource: [
            $class: "ConditionalStatusResultSource",
            results: [[$class: "AnyBuildResult", message: message, state: state]]
        ]
    ]);
}

pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
metadata:
  namespace: jenkins
  labels:
    jenkins/agent-type: kaniko
spec:
  nodeSelector:
    nodetype: agent
  containers:
  - name: jnlp
    image: chauid/jenkins-inbound-agent:jdk17-node22-k8s
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    command:
      - /busybox/cat
    tty: true
    volumeMounts:
      - name: docker-secret
        mountPath: /kaniko/.docker
  volumes:
    - name: docker-secret
      secret:
        secretName: docker-config-postsmith-hub
            '''
        }
    }

    options {
        timeout(time: 40, unit: 'SECONDS')
    }

    stages {
        stage('Init') {
            steps {
                script {
                    env.STAGE_NUMBER = 0
                    env.IMAGE_NAME = 'postsmith-hub.kr.ncr.ntruss.com/nexttt'
                    env.IMAGE_TAG = build.getProjectVersion('nodejs')
                }
            }
        }
        stage('Build npm') {
            steps {
                script {
                    setBuildStatus("Building Next.JS application", "CI / npm build", "PENDING")
                    env.STAGE_NUMBER = 1
                    build.npm()
                    setBuildStatus("Next.JS application built successfully", "CI / npm build", "SUCCESS")
                }
            }
        }
        stage('Build Docker Image') {
            when {
                anyOf { branch 'main'; branch 'PR-*' }
            }
            steps {
                script {
                    setBuildStatus("Building Docker image", "CI / Docker Build", "PENDING")
                    env.STAGE_NUMBER = 2
                    build.image(env.IMAGE_NAME, env.IMAGE_TAG, true)
                    setBuildStatus("Docker image built successfully", "CI / Docker Build", "SUCCESS")
                }
            }
        }
        stage('Deploy K8s') {
            when {
                anyOf { branch 'main'; branch 'PR-*' }
            }
            steps {
                script {
                    setBuildStatus("Deploy to Kubernetes cluster", "CD / Kubernetes rollout", "PENDING")
                    env.STAGE_NUMBER = 3
                    k8s.deploy("nexttt-app-deploy", "nexttt-app", "default", env.IMAGE_NAME, env.IMAGE_TAG)
                    setBuildStatus("Kubernetes cluster Deployed successfully", "CD / Kubernetes rollout", "SUCCESS")
                }
            }
        }

    }
    post {
        unsuccessful {
            script {
                switch (env.STAGE_NUMBER) {
                    case '0':
                        setBuildStatus("Failed to initialize the build process.", "Jenkins", "FAILURE")
                        break
                    case '1':
                        setBuildStatus("Failed to build the Next.JS application.", "CI / npm build", "FAILURE")
                        break
                    case '2':
                        setBuildStatus("Failed to build the Docker image.", "CI / Docker Build", "FAILURE")
                        break
                    case '3':
                        setBuildStatus("Failed to deploy to Kubernetes cluster.", "CD / Kubernetes rollout", "FAILURE")
                        break
                }
            }
        }
    }
}