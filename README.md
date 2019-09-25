#!groovy
@Library('itss-shared-lib@master') _

def CF = [
        ORG         : "mdtw",
        CF_URL      : "api.dev.sys.cs.sgp.dbs.com",
        SPACE       : "dev",
        CREDENTIAL  : "71d10f3a-5f56-49a9-8c3e-0c7d8bae3ce0",
        SERVICE_NAME: "DBTW_WEB",
        groupIdUrl  : "com/dbs/dbtw",
        nexusUrl    : "https://nexuscimgmt.sgp.dbs.com:8443/nexus",
        repository  : "DBTW"
]
def PIPELINE_NAME = "ARTIFACT PIPELINE"
def STAGE_NAME = ""
def gitInfo = {}

pipeline {
    agent {
        kubernetes {
            label env.BUILD_TAG
            defaultContainer 'jnlp'
            yaml """
spec:
  containers:
  - name: jnlp
    image: 172.30.150.100:5000/itss-dcif/ocpjenkins-nodejs-pcf:8.9.4
    imagePullPolicy: Always
    resources:
      requests:
        cpu: 1
        memory: 2Gi
      limits:
        cpu: 2
        memory: 6Gi
    tty: true
    workingDir: /var/lib/jenkins
"""
        }
    }
    options {
        skipDefaultCheckout()
        timestamps()
    }
    stages {
        stage('Clone') {
            steps {
                script {
                    STAGE_NAME = "Clone"
                    gitInfo = checkout scm
                    env.GIT_COMMIT = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
                }
            }
        }
        stage('Installing Dependencies') {
            steps {
                script {
                    STAGE_NAME = "Installing Dependencies"
                    npmInstallWithSass()
                    sh "npm run npm-clean || true"
                }
            }
        }
        stage('Lint Code check') {
            steps {
                script {
                    STAGE_NAME = "Linting"
                    if (!isHotBranch(gitInfo)) {
                        npmRun('lint:code')
                    }
                }
            }
        }
        stage('Build and Zip') {
            steps {
                script {
                    STAGE_NAME = "Build"
                    if (gitInfo.GIT_BRANCH.contains("master")) {
                        sh "export BRANCH_NAME=digitw && npm run build-prod"
                    }
                    else if (gitInfo.GIT_BRANCH.contains("release-")) {
                        sh "export BRANCH_NAME=digitw && npm run build-uat"
                    } else {
                        sh "export BRANCH_NAME=digitw && npm run build-test"
                    }
                    def exec = """
					rm -rf  ${WORKSPACE}/${env.BRANCH_NAME} && mkdir -p ${WORKSPACE}/${env.BRANCH_NAME}
                    cp nginx.conf dist/
                    cp -R dist/ ${env.BRANCH_NAME}
                    zip -r ${env.BRANCH_NAME}_${env.BUILD_NUMBER}.zip ${env.BRANCH_NAME}
					"""
                    sh exec
                }
            }
        }
        stage('Unit Test') {
            steps {
                script {
                    STAGE_NAME = "Unit Test"
                    sh 'export NODE_ENV=test'
                    if (!isHotBranch(gitInfo)) {
                        npmRun('test:update')
                        publish('Code Coverage Report', 'coverage/lcov-report')
                    }
                }
            }
        }
        stage('Static Analysis: Sonar scan') {
            steps {
                script {
                    STAGE_NAME = "Static Analysis"
                    Map mp = [commitID: gitInfo.GIT_COMMIT,
                              branch  : gitInfo.GIT_BRANCH,
                              repourl : gitInfo.GIT_URL,
                    ]
                    if (!isHotBranch(gitInfo)) {
                        performSonarScan(mp)
                    }
                }
            }
        }
        stage('Static Analysis: Fortify scan') {
            steps {
                script {
                    STAGE_NAME = "Fortify Scan"
                    Map scmmap = [commitID          : gitInfo.GIT_COMMIT,
                                  branch            : gitInfo.GIT_BRANCH,
                                  repourl           : gitInfo.GIT_URL,
                                  fortifyProjectName: "DBTW",
                                  fortifyVersionName: "MOD_dbtw-web",
                                  buildType         : "others"]
                    if (!isHotBranch(gitInfo)) {
                        performFortifyScan(scmmap)
                    }
                }
            }
        }
        stage("Nexus IQ scan") {
            steps {
                script {
                    STAGE_NAME = "Nexus IQ Scan"
                    if (!isHotBranch(gitInfo)) {
                        nexusPolicyEvaluation failBuildOnNetworkError: false, iqApplication: 'DBTW_MOD_dbtw-web', iqScanPatterns: [[scanPattern: '*.js'], [scanPattern: '*.jsx']], iqStage: 'build'
                    }
                }
            }
        }
        stage('Create Deployfile') {
            steps {
                script {
                    STAGE_NAME = "Create Deployfile"
                    if (gitInfo.GIT_BRANCH.contains("release-") || gitInfo.GIT_BRANCH.contains("integration") || gitInfo.GIT_BRANCH.contains("master")) {
                        version = env.BRANCH_NAME + "-" + env.BUILD_NUMBER
                        sh 'pwd'
                        def exec = """
cat <<'EOF' > deploy.json
{
    "branch"        : ${env.BRANCH_NAME},
    "build"         : ${env.BUILD_NUMBER},
    "url"           : ${CF.nexusUrl}/repository/${CF.repository}/${CF.groupIdUrl}/${CF.SERVICE_NAME}/${version}/${CF.SERVICE_NAME}-${version}.zip,
    "automationrun" : "false"
}
EOF
                        """
                        sh exec
                        sh "cat deploy.json"
                    } else {
                        println "Deployfile will be created for master, integration and release branch"
                    }
                }
            }
        }
        stage('Publish') {
            steps {
                script {
                    if (gitInfo.GIT_BRANCH.contains("release-") || gitInfo.GIT_BRANCH.contains("integration") || gitInfo.GIT_BRANCH.contains("master")) {
                        nexusArtifactUploader artifacts: [
                                [artifactId: CF.SERVICE_NAME, file: "./${env.BRANCH_NAME}_${env.BUILD_NUMBER}.zip", type: 'zip']
                        ],
                                credentialsId: 'nexusArtifactUploader',
                                groupId: 'com.dbs.dbtw',
                                nexusUrl: 'nexuscimgmt.sgp.dbs.com:8443/nexus',
                                nexusVersion: 'nexus3',
                                protocol: 'https',
                                repository: CF.repository,
                                version: env.BRANCH_NAME + "-" + gitInfo.GIT_COMMIT + "-" + env.BUILD_NUMBER
                    } else {
                        println "Publish will be performed for master, integration and release branch"
                    }
                }
            }
        }
        stage('Deploy') {
            steps {
                script {
                    STAGE_NAME = "Deploy to PCF"
                    if (gitInfo.GIT_BRANCH.contains("integration")) {
                        loginWithCred(CF)
                        deploy(CF)
                    } else {
                        println "Publish will be perform in integration or release branch"
                    }
                }
            }
        }
        stage('Outofband Deploy') {
            steps {
                script {
                    print "In manual mode"
                    if (gitInfo.GIT_BRANCH.contains("release-")) {
                        build job: '../Deploy/DBTW_Deploy', wait: true, parameters: [[$class: 'StringParameterValue', name: 'deployEnvironment', value: 'DBTWUAT'], [$class: 'StringParameterValue', name: 'CommitID', value: env.GIT_COMMIT], [$class: 'StringParameterValue', name: 'BUILD_NUMBER', value: env.BUILD_NUMBER], [$class: 'StringParameterValue', name: 'branchName', value: env.BRANCH_NAME], [$class: 'StringParameterValue', name: 'artefactVersion', value: env.BRANCH_NAME + "-" + gitInfo.GIT_COMMIT + "-" + env.BUILD_NUMBER]]
                        sh """
                        git tag ${gitInfo.GIT_BRANCH}
                        git --no-pager show ${gitInfo.GIT_BRANCH} || echo 'There is probably nothing to show.'
                        git push --tags || echo 'Tagging failed. There is probably already tagged.'
                        """
                    } else {
                        println "Deploy will be perform in integration or release branch"
                    }
                }
            }
        }
    }

    post {
        always {
            echo "global environment variables"
            echo "BRANCH_NAME: $BRANCH_NAME"
            echo "CHANGE_ID: $env.CHANGE_ID"
            echo "CHANGE_URL: $env.CHANGE_URL"
            echo "CHANGE_TITLE: $env.CHANGE_TITLE"
            postResult(PIPELINE_NAME, gitInfo, STAGE_NAME)
        }
        success {
            echo 'This will run only if successful'
        }
        failure {
            echo 'This will run only if failed'
        }
        unstable {
            echo 'This will run only if the run was marked as unstable'
        }
        changed {
            echo "Pipeline state has changed from $currentBuild.previousBuild"
        }
    }
}

def isHotBranch(gitInfo) {
    if (gitInfo.GIT_BRANCH.contains("hot-")) {
        println "Nexus will not be perform in hot branch"
        return true
    }
    return false
}

def postResult(PIPELINE_NAME, gitInfo, STAGE_NAME) {
    def jsonData = ""
    if (currentBuild.currentResult == 'SUCCESS' || '') {
        jsonData = """{ \"state\": \"SUCCESSFUL\",
        \"key\": \"$PIPELINE_NAME\",
        \"name\": \"$PIPELINE_NAME\",
        \"url\": \"$JOB_URL\",
        \"description\": \"Build Successful in $currentBuild.duration seconds\"  }"""
    } else {
        jsonData = """{ \"state\": \"FAILED\",
          \"key\": \"$PIPELINE_NAME\",
          \"name\": \"$PIPELINE_NAME\",
          \"url\": \"$JOB_URL\",
          \"description\": \"Build failed during $STAGE_NAME\"  }"""
    }

    echo "$jsonData"
    httpRequest contentType: 'APPLICATION_JSON',
            httpMode: 'POST',
            requestBody: jsonData,
            ignoreSslErrors: true,
            // for Prod Bitbucket
            authentication: "b908cf13-35bb-4e2e-8781-56959264d385",
            url: """https://bitbucket.sgp.dbs.com:8443/dcifgit/rest/build-status/1.0/commits/$gitInfo.GIT_COMMIT"""
}

def loginWithCred(CF) {
    withCredentials([[$class          : 'UsernamePasswordMultiBinding',
                      credentialsId   : "${CF.CREDENTIAL}",
                      usernameVariable: "USERNAME",
                      passwordVariable: "PASSWORD"]]) {
        echo "Credential used: ${CF.CREDENTIAL}"
        sh "set +x"
        sh "cf login -a ${CF.CF_URL} -u ${env.USERNAME} -p ${env.PASSWORD} -o ${CF.ORG} -s ${CF.SPACE} --skip-ssl-validation"
    }
    echo "before checking orgspace"
    orgspace = "-o ${CF.ORG} -s ${CF.SPACE}"
    sh "cf target ${orgspace}"
}

def deploy(CF) {
    try {
        // sh "unzip -o './${env.BRANCH_NAME}_${env.BUILD_NUMBER}.zip'"
        // sh "ls -lrt"
        sh "cf push"
    } catch (e) {
        echo " unexpected error ${e}"
        sh "cf logs --recent"
        error "Deployment Failed"
    }
}

def publish(name, dirName, indexFileName = 'index.html') {
    publishHTML target: [
            allowMissing         : false,
            alwaysLinkToLastBuild: false,
            keepAll              : true,
            reportDir            : dirName,
            reportFiles          : indexFileName,
            reportName           : name
    ]
}

def npmInstallWithSass() {
    sh "npm cache clear --force && echo sass_binary_site=https://bitbucket.sgp.dbs.com:8443/dcifgit/projects/WEBSTUDIO/repos/ext-libraries/raw > .npmrc && npm install && rm -f .npmrc"
}

def npmRun(command = "-v") {
    sh "npm run ${command}"
}
