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

    stages {
        stage('Init') {
            steps {
                script {
                    env.IMAGE_NAME = 'postsmith-hub.kr.ncr.ntruss.com/nexttt'
                    env.IMAGE_TAG = build.getProjectVersion('nodejs')
                    echo "1Deploy tag set to: ${env.IMAGE_TAG}"
                }
            }
        }
        stage('Build npm') {
            steps {
                script {
                    setBuildStatus("Building Next.JS application", "CI / npm build", "PENDING")
                    build.npm()
                    setBuildStatus("Next.JS application built successfully", "CI / npm build", "SUCCESS")
                }
            }
        }
        stage('Build Docker Image') {
            steps {
                script {
                    setBuildStatus("Building Docker image", "CI / Docker Build", "PENDING")
                    build.image(env.IMAGE_NAME, env.IMAGE_TAG, true)
                    setBuildStatus("Docker image built successfully", "CI / Docker Build", "SUCCESS")
                }
            }
        }
        stage('Deploy K8s') {
            steps {
                script {
                    echo "${env.BRANCH_NAME}"
                    setBuildStatus("Deploying to Kubernetes cluster", "CD / Kubernetes rollout", "PENDING")
                    k8s()
                    k8s.deploy("nexttt-app-deploy", "nexttt-app", "default", env.IMAGE_NAME, env.IMAGE_TAG)
                    setBuildStatus("Deployment to Kubernetes cluster completed successfully", "CD / Kubernetes rollout", "SUCCESS")
                }
            }
        }

    }
    post {
        aborted {
            setBuildStatus("The build was aborted by the user.", "Jenkins", "FAILURE")
            setBuildStatus("skip", "Jenkins11", "SKIPPED")
        }
        failure {
            setBuildStatus("Something went wrong during the build process. Please check the logs for details.", "Jenkins", "FAILURE")
        }
    }
}