#!groovy




void setBuildStatus(String message, String state) {
  step([
      $class: "GitHubCommitStatusSetter",
      reposSource: [$class: "ManuallyEnteredRepositorySource", url: "https://github.com/lneuhaus/pyrpl"],
      contextSource: [$class: "ManuallyEnteredCommitContextSource", context: "ci/jenkins/build-status"],
      errorHandlers: [[$class: "ChangingBuildStatusErrorHandler", result: "UNSTABLE"]],
      statusResultSource: [ $class: "ConditionalStatusResultSource", results: [[$class: "AnyBuildResult", message: message, state: state]] ]
  ]);
}

def getRepoURL() {
  sh "git config --get remote.origin.url > .git/remote-url"
  return readFile(".git/remote-url").trim()
}

def getCommitSha() {
  sh "git rev-parse HEAD > .git/current-commit"
  return readFile(".git/current-commit").trim()
}

def updateGithubCommitStatus(build) {
  // workaround https://issues.jenkins-ci.org/browse/JENKINS-38674
  repoUrl = getRepoURL()
  commitSha = getCommitSha()

  step([
    $class: 'GitHubCommitStatusSetter',
    reposSource: [$class: "ManuallyEnteredRepositorySource", url: repoUrl],
    commitShaSource: [$class: "ManuallyEnteredShaSource", sha: commitSha],
    errorHandlers: [[$class: 'ShallowAnyErrorHandler']],
    statusResultSource: [
      $class: 'ConditionalStatusResultSource',
      results: [
        [$class: 'BetterThanOrEqualBuildResult', result: 'SUCCESS', state: 'SUCCESS', message: build.description],
        [$class: 'BetterThanOrEqualBuildResult', result: 'FAILURE', state: 'FAILURE', message: build.description],
        [$class: 'AnyBuildResult', state: 'FAILURE', message: 'Loophole']
      ]
    ]
  ])
}

pipeline {
    triggers { pollSCM('*/1 * * * *') }

    options {
        // skipDefaultCheckout(true)  // rather do the checkout in all stages
        // Keep the 10 most recent builds
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
        // lock the redpitaya such that no two pipelines running in parallel can interfere
    }


    environment {
        REDPITAYA_HOSTNAME = "192.168.178.26"
        //REDPITAYA_HOSTNAME = "rp-f03f3a"
        //REDPITAYA_HOSTNAME = "nobody.justdied.com"
        REDPITAYA_USER = "root"
        REDPITAYA_PASSWORD = "Kartoffelschmarn"
        DOCKER_ARGS = '-u root -v /tmp/.X11-unix:/tmp/.X11-unix -e DISPLAY=:0 --net=host'
        //NOSETESTS_COMMAND = 'nosetests pyrpl/test/test_ipython_notebook/test_ipython_kernel.py'
        NOSETESTS_COMMAND = 'nosetests'
    }

    agent any

    stages {
        stage('Notify github of build start') {
            agent any
            steps { setBuildStatus("Build started...", "PENDING") }}
        stage('Unit tests') { parallel {
            stage('Python 3.7') {
                agent { dockerfile { args "$DOCKER_ARGS"
                                     additionalBuildArgs  '--build-arg PYTHON_VERSION=3.7' }}
                steps {
                    lock('redpitaya')
                    sh  ''' which python
                            python -V
                            echo $PYTHON_VERSION
                            # use a custom global configfile adapted to the hardware for unit tests
                            cp ./jenkins_global_config.yml ./pyrpl/config/global_config.yml
                            python setup.py install
                        '''
                    sh "$NOSETESTS_COMMAND"
                    unlock('redpitaya'}
                post { always { junit allowEmptyResults: true, testResults: 'unit_test_results.xml' }}}
            stage('Python 3.6') {
                agent { dockerfile { args "$DOCKER_ARGS"
                                     additionalBuildArgs  '--build-arg PYTHON_VERSION=3.6' }}
                steps {
                    sh  ''' which python
                            python -V
                            echo $PYTHON_VERSION
                            # use a custom global configfile adapted to the hardware for unit tests
                            cp ./jenkins_global_config.yml ./pyrpl/config/global_config.yml
                            python setup.py install
                        '''
                    sh "$NOSETESTS_COMMAND"}
                post { always { junit allowEmptyResults: true, testResults: 'unit_test_results.xml' }}}
            /*stage('Python 3.5') {
                agent { dockerfile { args "$DOCKER_ARGS"
                                     additionalBuildArgs  '--build-arg PYTHON_VERSION=3.5' }}
                steps {
                    sh  ''' which python
                            python -V
                            echo $PYTHON_VERSION
                            # use a custom global configfile adapted to the hardware for unit tests
                            cp ./jenkins_global_config.yml ./pyrpl/config/global_config.yml
                            python setup.py install
                        '''
                    sh "$NOSETESTS_COMMAND"}
                post { always { junit allowEmptyResults: true, testResults: 'unit_test_results.xml' }}}*/
            stage('Python 2.7') {
                agent { dockerfile { args "$DOCKER_ARGS"
                                     additionalBuildArgs  '--build-arg PYTHON_VERSION=2.7' }}
                steps {
                    sh  ''' which python
                            python -V
                            echo $PYTHON_VERSION
                            # use a custom global configfile adapted to the hardware for unit tests
                            cp ./jenkins_global_config.yml ./pyrpl/config/global_config.yml
                            python setup.py install
                        '''
                    sh "$NOSETESTS_COMMAND"}
                post { always { junit allowEmptyResults: true, testResults: 'unit_test_results.xml' }}}
        }}

        stage('Build and deploy package') {
            agent { dockerfile { args '-u root -v /tmp/.X11-unix:/tmp/.X11-unix -e DISPLAY=:0 --net=host'
                         additionalBuildArgs  '--build-arg PYTHON_VERSION=3.6' }}
            when {
                expression { currentBuild.result == null || currentBuild.result == 'SUCCESS'}}
            steps {
                sh  ''' python setup.py install
                        python setup.py bdist_wheel
                        # twine upload dist/*
                    '''
                sh  ''' pip install pyinstaller
                        pyinstaller pyrpl.spec
                        mv dist/pyrpl ./pyrpl-linux-develop
                    '''
                //sh 'python .deploy_to_sourceforge.py pyrpl-linux-develop'
                }
            post { always { archiveArtifacts allowEmptyArchive: true, artifacts: 'dist/*whl, pyrpl-linux-develop', fingerprint: true}}}}
        post {
            failure {
                emailext (
                    attachLog: true,
                    subject: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                    body: """<p>FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]':</p>
                             <p>Check console output at <a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a></p>""",
                    compressLog: false,
                    recipientProviders: [requestor(), developers(), brokenTestsSuspects(), brokenBuildSuspects(), upstreamDevelopers(), culprits()],
                    replyTo: 'pyrpl.readthedocs.io@gmail.com',
                    to: 'pyrpl.readthedocs.io@gmail.com')
                setBuildStatus("Build failed!", "FAILURE")
                }
            success { setBuildStatus("Build successful!", "SUCCESS") }
            unstable { setBuildStatus("Build erroneous!", "ERROR") }
        }
}

