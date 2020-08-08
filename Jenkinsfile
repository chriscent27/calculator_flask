def getRepoURL() {
  sh "git config --get remote.origin.url > .git/remote-url"
  return readFile(".git/remote-url").trim()
}
def getCommitSha() {
  sh "git rev-parse HEAD > .git/current-commit"
  return readFile(".git/current-commit").trim()
}
void setBuildStatus(String message, String state) {
  step([
      $class: "GitHubCommitStatusSetter",
      reposSource: [$class: "ManuallyEnteredRepositorySource", url: getRepoURL()],
      commitShaSource: [$class: "ManuallyEnteredShaSource", sha: commitSha],
      errorHandlers: [[$class: "ChangingBuildStatusErrorHandler", result: "UNSTABLE"]],
      statusResultSource: [ $class: "ConditionalStatusResultSource", results: [[$class: "AnyBuildResult", message: message, state: state]] ]
  ]);
}
pipeline {
    agent none
    stages {
        stage('Start') {
            agent any
            steps {
                setBuildStatus("In progress","PENDING")
            }

        }
        stage('Build') {
            agent {
                docker {
                    image 'python:3.8.5-alpine3.11'
                }
            }
            steps {
                sh 'python -m py_compile calculator.py'
                stash(name: 'compiled-results', includes: '*.py*')
            }
        }
        stage('Test') {
            agent {
                docker {
                    image 'qnib/pytest'
                }
            }
            steps {
                sh 'py.test --junit-xml test-reports/results.xml test_calculator.py'
            }
            post {
                always {
                    junit 'test-reports/results.xml'
                }
            }
        }
        stage('Deliver') {
            agent any
            environment {

                VOLUME = '$(pwd):/src'
                IMAGE = 'cdrx/pyinstaller-linux:python3'
            }
            steps {
                dir(path: env.BUILD_ID) {
                    unstash(name: 'compiled-results')
                    sh "docker run --rm -v ${VOLUME} ${IMAGE} 'pyinstaller -F calculator.py'"
                }
            }
            post {
                success {
                    archiveArtifacts "${env.BUILD_ID}/dist/calculator"
                    sh "docker run --rm -v ${VOLUME} ${IMAGE} 'rm -rf build dist'"
                    setBuildStatus("Build complete", "SUCCESS")
                }
                failure {
                setBuildStatus("Build failed", "FAILURE")
                }
            }
        }
    }
}
