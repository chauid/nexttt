@Library('my-shared-library') _

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
        timeout(5)
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
                    github.setCommitStatus("Building Next.JS application", "CI / npm build", "PENDING")
                    env.STAGE_NUMBER = 1
                    build.npm()
                    github.setCommitStatus("Next.JS application built successfully", "CI / npm build", "SUCCESS")
                }
            }
        }
        stage('Build Container Image') {
            when {
                anyOf { branch 'main'; branch 'PR-*' }
            }
            steps {
                script {
                    github.setCommitStatus("Building Container image", "CI / Image Build", "PENDING")
                    env.STAGE_NUMBER = 2
                    build.image(env.IMAGE_NAME, env.IMAGE_TAG, true)
                    github.setCommitStatus("Container image built successfully", "CI / Image Build", "SUCCESS")
                }
            }
        }
        stage('Deploy K8s') {
            when {
                anyOf { branch 'main'; branch 'PR-*' }
            }
            steps {
                script {
                    github.setCommitStatus("Deploy to Kubernetes cluster", "CD / Kubernetes rollout", "PENDING")
                    env.STAGE_NUMBER = 3
                    k8s.deploy("nexttt-app-deploy", "nexttt-app", "default", env.IMAGE_NAME, env.IMAGE_TAG)
                    github.setCommitStatus("Kubernetes cluster Deployed successfully", "CD / Kubernetes rollout", "SUCCESS")
                }
            }
        }

    }
    post {
        unsuccessful {
            script {
                switch (env.STAGE_NUMBER) {
                    case '0':
                        github.setCommitStatus("Failed to initialize the build process.", "Jenkins", "FAILURE")
                        break
                    case '1':
                        github.setCommitStatus("Failed to build the Next.JS application.", "CI / npm build", "FAILURE")
                        break
                    case '2':
                        github.setCommitStatus("Failed to build the Container image.", "CI / Image Build", "FAILURE")
                        break
                    case '3':
                        github.setCommitStatus("Failed to deploy to Kubernetes cluster.", "CD / Kubernetes rollout", "FAILURE")
                        break
                }
            }
        }
    }
}