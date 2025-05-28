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
    image: chauid/jenkins-inbound-agent:1.0
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
                env.IMAGE_NAME = 'postsmith-hub.kr.ncr.ntruss.com/nexttt'
                env.IMAGE_TAG = build.getProjectVersion('nodejs')
                echo "Deploy tag set to: ${env.IMAGE_TAG}"
            }
        }
        stage('Build Node.js') {
            steps {
                build.npm()
            }
        }
        stage('Build Docker Image') {
            steps {
                build.image(env.IMAGE_NAME, env.IMAGE_TAG, true)
            }
        }
    }
}