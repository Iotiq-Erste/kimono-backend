def prBranchStages = [
    'Compile',
    'Analyze Code'
//     'Test',
]

def deployStages = [
    'Compile',
    'Analyze Code',
//     'Test',
    'Deploy'
]

stagesMap = []

pipeline {
    agent any

    environment {
        DEPLOYMENT_BRANCH          = "master"
        DEPLOYMENT_SERVER_HOSTNAME = "fileserver.dakikyazilim.com"
        DEPLOYMENT_SERVER_USERNAME = "erste"
        DOCKER_COMPOSE_FILE_PATH   = "dockerfiles/kimono/backend/docker-compose.yml"
    }

    tools {
        jdk 'jdk-21'
    }

    stages {
        stage("Set Stages") {
            steps {
                script{
                    lastStage = env.STAGE_NAME
                    env.IS_PR = "${env.CHANGE_ID ? true : false}"
                    env.DEPLOY = "${env.BRANCH_NAME == env.DEPLOYMENT_BRANCH ? true : false}"
                    env.COMMIT_MESSAGE = sh(script: "git log --pretty=short -1 | cat", returnStdout: true).trim()
                    if(env.DEPLOY.toBoolean()){
                        stagesMap = deployStages
                    }else{
                        stagesMap = prBranchStages
                    }
                    stagesMap.each { value ->
                        publishChecks name: "${value}", status: 'QUEUED'
                    }
                    env.TRIGGERED_BY_SUBMODULE = "${params.SUBMODULE_JOB_NAME != null ? true : false}"
                    if(env.TRIGGERED_BY_SUBMODULE.toBoolean()){
                        env.SUBMODULE_JOB_NAME = params.SUBMODULE_JOB_NAME
                        env.SUBMODULE_COMMIT_MESSAGE = params.SUBMODULE_COMMIT_MESSAGE
                        env.SUBMODULE_BUILD_DISPLAY_NAME = params.SUBMODULE_BUILD_DISPLAY_NAME
                        env.SUBMODULE_RUN_DISPLAY_URL = params.SUBMODULE_RUN_DISPLAY_URL
                    }
                }
                sh 'printenv'
            }
        }

        stage("Compile") {
            steps {
                script {
                    lastStage = env.STAGE_NAME

                }
                publishChecks name: 'Compile', status: 'IN_PROGRESS'
                echo 'compiling the project...'

                sh './mvnw clean'
                sh './mvnw compile'
                sh './mvnw install -Dmaven.test.skip=true'

                publishChecks name: 'Compile', title: 'Compile', summary: 'mvnw compile',
                    text: 'running ./mvnw compile',
                    detailsURL: '',
                    actions: [[label:'mvnw', description:'compile', identifier:'mvnw']]
                script { stagesMap.remove(0) }
            }
        }
        stage("Analyze Code") {
            steps {
                script { lastStage = env.STAGE_NAME }
                publishChecks name: 'Analyze Code', status: 'IN_PROGRESS'
                echo 'Code analyzing started..'

                withSonarQubeEnv(installationName: 'SonarIotiqDev') {
                    sh './mvnw sonar:sonar'
                }

                publishChecks name: 'Analyze Code'
                script { stagesMap.remove(0) }
            }
        }
//         stage("test") {
//             steps {
//                 script { failedStage = env.STAGE_NAME }
//                 publishChecks name: 'Test', status: 'IN_PROGRESS'
//                 echo 'executing tests...'
//                 sh './mvnw test --scan'
//                 publishChecks name: 'Test'
//                 script { stagesMap.remove(0) }
//             }
//         }
        stage("Deploy") {
            when {
                allOf {
                    branch env.DEPLOYMENT_BRANCH
                }
            }
            steps {
                script { lastStage = env.STAGE_NAME }
                publishChecks name: 'Deploy', status: 'IN_PROGRESS'
                echo 'pushing docker image to docker hub registry... (iotiqdevops account)'
                dir('application') {
                    sh './mvnw jib:build'
                }
                sshagent(credentials: ['ersteprivkey']) {
                    sh "ssh -o StrictHostKeyChecking=no -l ${DEPLOYMENT_SERVER_USERNAME} ${DEPLOYMENT_SERVER_HOSTNAME} 'docker-compose -f ${DOCKER_COMPOSE_FILE_PATH} pull'"
                    sh "ssh -o StrictHostKeyChecking=no -l ${DEPLOYMENT_SERVER_USERNAME} ${DEPLOYMENT_SERVER_HOSTNAME} 'docker-compose -f ${DOCKER_COMPOSE_FILE_PATH} up -d'"
                }
                publishChecks name: 'Deploy'
            }
        }
    }
    post {
        always {
            script {
                env.PARENT_JOB_NAME = "${JOB_NAME.substring(0, JOB_NAME.indexOf('/'))}"
            }
        }
        success {
            script {
                if(env.DEPLOY.toBoolean()){
                    DISCORD_MESSAGE_TITLE = ":star: :rocket: *${PARENT_JOB_NAME}* ${BUILD_DISPLAY_NAME} "
                    DISCORD_MESSAGE_DESCRIPTION = "Branch/PR:       *${env.GIT_BRANCH}*\n\n"
                    if (env.IS_PR.toBoolean()) {
                        DISCORD_MESSAGE_DESCRIPTION += "Pull Request: [${CHANGE_URL}] (${CHANGE_TITLE})" +
                                         "   ( ${CHANGE_BRANCH}   :arrow_right:   ${CHANGE_TARGET} )\n\n"
                    }
                    if (env.TRIGGERED_BY_SUBMODULE.toBoolean()) {
                       DISCORD_MESSAGE_DESCRIPTION += "Triggered By: [${env.SUBMODULE_JOB_NAME} ${env.SUBMODULE_BUILD_DISPLAY_NAME}](${SUBMODULE_RUN_DISPLAY_URL})\n\n"
                       DISCORD_MESSAGE_DESCRIPTION += env.SUBMODULE_COMMIT_MESSAGE
                    }
                    else{
                        DISCORD_MESSAGE_DESCRIPTION += env.COMMIT_MESSAGE
                    }
                    discordSend description: DISCORD_MESSAGE_DESCRIPTION,
                        footer: null,
                        link: RUN_DISPLAY_URL,
                        result: "SUCCESS",
                        title: DISCORD_MESSAGE_TITLE,
                        webhookURL: DISCORD_KIMONO_WEBHOOK_URL
                }
            }
        }
        failure {
            script {
                stagesMap.each { value ->
                    publishChecks name: "${value}", conclusion: 'CANCELED', status: 'COMPLETED'
                }
                if(env.DEPLOY.toBoolean() || env.IS_PR.toBoolean()){
                    DISCORD_MESSAGE_TITLE = ":x: *${PARENT_JOB_NAME}*  ${BUILD_DISPLAY_NAME} :x:"
                    DISCORD_MESSAGE_DESCRIPTION = "Branch/PR:         *${env.GIT_BRANCH}* "+
                                                  "Stage:          *${lastStage}* \n\n"
                    if (env.IS_PR.toBoolean()) {
                        DISCORD_MESSAGE_DESCRIPTION += "Pull Request: [${CHANGE_URL}] (${CHANGE_TITLE})" +
                                         "   ( ${CHANGE_BRANCH}   :arrow_right:   ${CHANGE_TARGET} )\n\n"
                    }
                    if (env.TRIGGERED_BY_SUBMODULE.toBoolean()) {
                       DISCORD_MESSAGE_DESCRIPTION += "Triggered By: [${env.SUBMODULE_JOB_NAME} ${env.SUBMODULE_BUILD_DISPLAY_NAME}](${SUBMODULE_RUN_DISPLAY_URL})\n\n"
                       DISCORD_MESSAGE_DESCRIPTION += env.SUBMODULE_COMMIT_MESSAGE
                    }
                    else{
                        DISCORD_MESSAGE_DESCRIPTION += env.COMMIT_MESSAGE
                    }
                    discordSend description: DISCORD_MESSAGE_DESCRIPTION,
                        footer: null,
                        link: RUN_DISPLAY_URL,
                        result: "FAILURE",
                        title: DISCORD_MESSAGE_TITLE,
                        webhookURL: DISCORD_KIMONO_WEBHOOK_URL
                }
            }
            publishChecks name: "${lastStage}", conclusion: 'FAILURE',  status: 'COMPLETED'
        }
    }
}
