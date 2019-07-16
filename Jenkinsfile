pipeline {
    agent any
    options {
        // 禁止同时运行多个流水线
        disableConcurrentBuilds()
    }
    environment {
        // 获取git提交信息
        GIT_COMMIT_USER = sh(script: 'git log -1 --pretty=format:"%an"', returnStdout: true).trim()
        GIT_COMMIT_HASH = sh(script: 'git log -1 --pretty=format:"%H"', returnStdout: true).trim()
        GIT_COMMIT_TITLE = sh(script: 'git log -1 --pretty=format:"%s"', returnStdout: true).trim()
        JENKINS_FILE_MD5 = sh(script: 'md5sum Jenkinsfile|cut -d " " -f 1', returnStdout: true).trim()
        //多分支的流水线使用
        PROJECT_NAME = sh(script: 'echo ${JOB_NAME%/*}', returnStdout: true).trim()


        USE_YUFABU = 'YES'           // YES启用预发布环境, NO不启用
        USE_PRODUCTION_1 = 'YES'     //  YES启用生产环境, NO不启用
        USE_PRODUCTION_2 = 'YES'     //  YES启用生产环境, NO不启用

        PRODUCTION_1_PROJECT_NAME = "jcn-ci-cd-test.xxxxxxe.com"     // 生产环境所用域名，如不同需要手动修改
        PRODUCTION_2_PROJECT_NAME = "${PROJECT_NAME}"    // 生产环境所用域名，如不同需要手动修改

        // 定义发送消息通知的用户
        qa_group = 'neowei'          // 测试
        ops_group = 'neowei'         // 运维
        dev_group = 'neowei'         // 开发
        boss_group = 'neowei'        // 发布审批

    }

    stages {
        stage('编译代码') {
            parallel {
                stage('via npm') {
                    steps {
                        sh 'cd frontend/web/source && npm install && npm run build'
                    }
                    post {
                        failure {
                            sh '${JENKINS_HOME}/ansible/msg.py -n  -u "${ops_group},${dev_group}" -T " ${JOB_NAME} Build error " -C "点击 ${RUN_DISPLAY_URL} 查看详情"'
                        }
                    }
                }
            }
        }

        stage('发布到QA环境'){
            steps {
                // 使用ansible发布QA环境
                ansiColor('xterm') {
                    ansiblePlaybook(
                        playbook: '${JENKINS_HOME}/ansible/site.yml',
                        inventory: '${JENKINS_HOME}/ansible/qatest',
                        disableHostKeyChecking: true,
                        //inventoryContent: 'master',
                        credentialsId: 'cbbbc2c3-74c2-46a8-91cd-185a08255a53',
                        colorized: true,
                        extras: '-e project_workspace="${WORKSPACE}" -e project_name="${PROJECT_NAME}" -e GIT_COMMIT_HASH="${GIT_COMMIT_HASH}" -e JENKINS_FILE_MD5="${JENKINS_FILE_MD5}"'
                    )
                } 
            }
            post {
                success {
                    // 通知QA同学进行测试
                    sh '${JENKINS_HOME}/ansible/msg.py -n -u "${qa_group},${dev_group}" -T "项目名: ${JOB_NAME} 已经发布到 QA环境" -C "版本HASH:  ${GIT_COMMIT_HASH}\n请安排测试。\n点击 ${RUN_DISPLAY_URL} 提交测试意见"'
                }
                failure {
                    // 发布失败通知
                    sh '${JENKINS_HOME}/ansible/msg.py -n -u "${ops_group},${dev_group}" -T "项目名: ${JOB_NAME} QA 环境发布失败" -C "点击 ${RUN_DISPLAY_URL} 查看详情"'
                }
            }
        }

        stage('QA环境测试'){
            steps {
                // 询问QA环境测试是否通过
                script {
                    try {
                        input(
                            id: 'askqatest', message: 'QA 环境测试是否通过？', ok:'通过并继续',submitter: 'qauser1'
                        )
                    } catch(err) { 
                        // 获取执行Input的用户
                        env.user = err.getCauses()[0].getUser().toString()
                        userInput = false
                        env.QATEST_TEST = false
                        echo "Aborted by: [${user}]"
                        // 发送终止通知
                        sh '${JENKINS_HOME}/ansible/msg.py -n -u "${qa_group},${dev_group}" -T "项目名: ${JOB_NAME} QA 环境测试不通过" -C "版本HASH:  ${GIT_COMMIT_HASH}\n测试不通过，发布流程取消"'
                        // 抛出异常确保流程终止
                        throw err
                    } 
                }
            }
        }

        stage('发布到预发布环境'){
            when {
                environment name: 'USE_YUFABU', value: 'YES'
            }

            steps {
                ansiColor('xterm') {
                    ansiblePlaybook(
                        playbook: '${JENKINS_HOME}/ansible/site.yml',
                        inventory: '${JENKINS_HOME}/ansible/yufabu',
                        disableHostKeyChecking: true,
                        //inventoryContent: 'master',
                        credentialsId: 'cbbbc2c3-74c2-46a8-91cd-185a08255a53',
                        colorized: true,
                        extras: '-e project_workspace="${WORKSPACE}" -e project_name="${PROJECT_NAME}" -e GIT_COMMIT_HASH="${GIT_COMMIT_HASH}" -e JENKINS_FILE_MD5="${JENKINS_FILE_MD5}"'
                    )
                } 
            }
            post {
                success {
                    // 通知QA同学进行测试
                    sh '${JENKINS_HOME}/ansible/msg.py -n -u "${qa_group},${dev_group}" -T "项目名: ${JOB_NAME} 已经发布到 预发布 环境" -C "版本HASH:  ${GIT_COMMIT_HASH}\n请安排测试。\n点击 ${RUN_DISPLAY_URL} 提交测试意见"'
                }
                failure {
                    // 发布失败通知
                    sh '${JENKINS_HOME}/ansible/msg.py -n -u "${ops_group},${dev_group}" -T "项目名: ${JOB_NAME} QA 环境发布失败" -C "点击 ${RUN_DISPLAY_URL} 查看详情"'
                }
            }
        }

        stage('预发布环境测试'){
            when {
                environment name: 'USE_YUFABU', value: 'YES'
            }
            steps {
                // 询问QA环境测试是否通过
                script {
                    try {
                        input (
                            id: 'askyufabu', message: '预发布 环境测试是否通过？', ok:'通过并继续',submitter: 'qauser1'
                        )
                    } catch(err) { 
                        // 获取执行Input的用户
                        env.user = err.getCauses()[0].getUser().toString()
                        userInput = false
                        env.YUFABU_TEST = false
                        echo "Aborted by: [${user}]"
                        // 发送终止通知
                        sh '${JENKINS_HOME}/ansible/msg.py -n -u "${qa_group},${dev_group}" -T "项目名: ${JOB_NAME} 预发布 环境测试不通过" -C "版本HASH:  ${GIT_COMMIT_HASH}\n测试不通过，发布流程取消"'
                        // 抛出异常确保流程终止
                        throw err
                    } 
                }
            }
        }
        

        // stage('生产环境发布审批'){
        //     steps {
        //         sh '${JENKINS_HOME}/ansible/msg.py -n -u "${boss_group}" -T "项目名: ${JOB_NAME} 环境测试已通过，请审批" -C "点击 ${RUN_DISPLAY_URL} 进行审批是否发布至 生产 环境"'
        //         script {
        //             try {
        //                 input (
        //                     id: 'deploytoproduction', message: '测试已通过，是否发布到 生产 环境',ok:'确认发布'
        //                 )
        //             } catch(err) { 
        //                 env.user = err.getCauses()[0].getUser().toString()
        //                 userInput = false
        //                 echo "Aborted by: [${user}]"
        //                 sh '${JENKINS_HOME}/ansible/msg.py -n -u "${qa_group},${dev_group}" -T "项目名: ${JOB_NAME} 审批不通过" -C "项目名称:  ${JOB_NAME}\n提交信息:  ${GIT_COMMIT_TITLE}\n提交用户:  ${GIT_COMMIT_USER}\n版本HASH:  ${GIT_COMMIT_HASH}\n本次发布被取消。"'
        //                 throw err
        //             }
        //         }
        //     }
        //     post {
        //         success {
        //             sh '${JENKINS_HOME}/ansible/msg.py -n -u "${ops_group}" -T "项目名: ${JOB_NAME} 审批通过" -C "项目名称:  ${JOB_NAME}\n提交信息:  ${GIT_COMMIT_TITLE}\n提交用户:  ${GIT_COMMIT_USER}\n版本HASH:  ${GIT_COMMIT_HASH}\n请运维进行发布至生产环境"'
        //         }
        //     }
        // }

        stage('生产环境发布') {
            // failFast true
            parallel {
                // stage('发布生产环境') {
                //     when {
                //         environment name: 'USE_PRODUCTION_1', value: 'YES'
                //     }
                //     steps {
                //         echo "${PRODUCTION_1_PROJECT_NAME}"
                //         script {
                //             try {
                //                 input (
                //                     id: 'deploytoproduction_1', message: '发布到 生产 环境',ok:'确认发布'
                //                 )
                //             } catch(err) { 
                //                 env.user = err.getCauses()[0].getUser().toString()
                //                 userInput = false
                //                 echo "Aborted by: [${user}]"
                //                 // 发送通知消息，告知取消发布
                //                 sh '${JENKINS_HOME}/ansible/msg.py -n -u "${ops_group},${dev_group}" -T "项目名: ${JOB_NAME} 生产 环境取消发布" -C "项目名称:  ${JOB_NAME}\n提交信息:  ${GIT_COMMIT_TITLE}\n提交用户:  ${GIT_COMMIT_USER}\n版本HASH:  ${GIT_COMMIT_HASH}\n本次发布被取消。"'
                //                 throw err
                //             }
                //         }
                        
                //         ansiColor('xterm') {
                //             ansiblePlaybook(
                //                 playbook: '${JENKINS_HOME}/ansible/site.yml',
                //                 inventory: '${JENKINS_HOME}/ansible/production',
                //                 disableHostKeyChecking: true,
                //                 //inventoryContent: 'master',
                //                 credentialsId: 'cbbbc2c3-74c2-46a8-91cd-185a08255a53',
                //                 colorized: true,

                //                 //TODO: 注意这个${PROJECT_NAME}， 如果两个生产环境的域名对应不上，要注意手动修改为正确的名字
                //                 extras: '-e project_workspace="${WORKSPACE}" -e project_name="${PRODUCTION_1_PROJECT_NAME}" -e GIT_COMMIT_HASH="${GIT_COMMIT_HASH}" -e JENKINS_FILE_MD5="${JENKINS_FILE_MD5}"'
                //             )
                //         }

                //         // ansible发布成功，通知QA同学进行测试
                //         sh '${JENKINS_HOME}/ansible/msg.py -n -u "${qa_group},${dev_group}" -T "项目名: ${JOB_NAME} 已经发布到 生产 环境" -C "版本HASH:  ${GIT_COMMIT_HASH}\n请安排测试。\n点击 ${RUN_DISPLAY_URL} 提交测试意见"'

                //         script {
                //             try {
                //                 input (
                //                     id: 'production_1_test', message: '生产环境测试是否通过?',ok:'测试通过',submitter: 'qauser1'
                //                 )
                //             } catch(err) { 
                //                 env.user = err.getCauses()[0].getUser().toString()
                //                 userInput = false

                //                 // 定义PRODUCTION_1_TEST，以便最后的post判断流程为何失败
                //                 env.PRODUCTION_1_TEST = false
                //                 echo "Aborted by: [${user}]"
                //                 sh '${JENKINS_HOME}/ansible/msg.py -n -u "${ops_group},${qa_group},${dev_group}" -T "项目名: ${JOB_NAME} 生产 环境测试不通过" -C "版本HASH:  ${GIT_COMMIT_HASH}\n测试不通过，需运维介入进行生产环境回滚"'
                //                 throw err
                //             } 
                //         }

                //     }
                  
                //     post {
                //         failure {
                //             script {
                //                 // 判断流程是否是由测试人员取消，若不是则发送消息通知告知发布失败
                //                 if( env.PRODUCTION_1_TEST != false ) {
                //                     sh '${JENKINS_HOME}/ansible/msg.py -n -u "${ops_group},${dev_group}" -T "项目名: ${JOB_NAME} 生产 环境发布失败" -C "点击 ${RUN_DISPLAY_URL} 查看详情"'
                //                 }
                //             } 
                //         }
                //     }
                // }



                stage('发布生产环境') {
                    when {
                        environment name: 'USE_PRODUCTION_2', value: 'YES'
                    }
                    steps {
                        echo "${PRODUCTION_2_PROJECT_NAME}"
                        script {
                            try {
                                input (
                                    id: 'deploytoproduction_2', message: '发布到 生产 环境',ok:'确认发布'
                                )
                            } catch(err) { 
                                env.user = err.getCauses()[0].getUser().toString()
                                userInput = false
                                echo "Aborted by: [${user}]"
                                // 发送通知消息，告知取消发布
                                sh '${JENKINS_HOME}/ansible/msg.py -n -u "${ops_group},${dev_group}" -T "项目名: ${JOB_NAME} 生产 环境取消发布" -C "项目名称:  ${JOB_NAME}\n提交信息:  ${GIT_COMMIT_TITLE}\n提交用户:  ${GIT_COMMIT_USER}\n版本HASH:  ${GIT_COMMIT_HASH}\n本次发布被取消。"'
                                throw err
                            }
                        }
                        ansiColor('xterm') {
                            ansiblePlaybook(
                                playbook: '${JENKINS_HOME}/ansible/site.yml',
                                inventory: '${JENKINS_HOME}/ansible/production',
                                disableHostKeyChecking: true,
                                //inventoryContent: 'master',
                                credentialsId: 'cbbbc2c3-74c2-46a8-91cd-185a08255a53',
                                colorized: true,

                                //TODO: 注意这个${PROJECT_NAME}， 如果两个生产环境的域名对应不上，要注意手动修改为正确的名字
                                extras: '-e project_workspace="${WORKSPACE}" -e project_name="${PRODUCTION_2_PROJECT_NAME}" -e GIT_COMMIT_HASH="${GIT_COMMIT_HASH}" -e JENKINS_FILE_MD5="${JENKINS_FILE_MD5}"'
                            )
                        }

                        // 通知QA同学进行测试
                        sh '${JENKINS_HOME}/ansible/msg.py -n -u "${qa_group},${dev_group}" -T "项目名: ${JOB_NAME} 已经发布到 生产 环境" -C "版本HASH:  ${GIT_COMMIT_HASH}\n请安排测试。\n点击 ${RUN_DISPLAY_URL} 提交测试意见"'

                        script {
                            try {
                                input (
                                    id: 'production_2_test', message: '生产环境测试是否通过?',ok:'测试通过',submitter: 'qauser1'
                                )
                            } catch(err) { 
                                env.user = err.getCauses()[0].getUser().toString()
                                userInput = false
                                env.PRODUCTION_2_TEST = false
                                echo "Aborted by: [${user}]"
                                sh '${JENKINS_HOME}/ansible/msg.py -n -u "${qa_group},${ops_group},${dev_group}" -T "项目名: ${JOB_NAME} 生产 环境测试不通过" -C "版本HASH:  ${GIT_COMMIT_HASH}\n测试不通过，需运维介入进行生产环境回滚"'
                                throw err
                            } 
                        }

                    }
                  
                    post {
                        // success{
                        //     sh '${JENKINS_HOME}/ansible/msg.sh "项目名: ${JOB_NAME} 已发布至 QA 环境" "项目名称:  ${JOB_NAME}\\n提交信息:  ${GIT_COMMIT_TITLE}\\n提交用户:  ${GIT_COMMIT_USER}\\n版本hash:  ${GIT_COMMIT_HASH}\\n\\n点击 ${RUN_DISPLAY_URL} 确认测试是否通过以便进入下一环节"'
                        // }
                        failure {
                            script {
                                if( env.PRODUCTION_2_TEST != false ) {
                                    sh '${JENKINS_HOME}/ansible/msg.py -n -u "${ops_group},${dev_group}" -T "项目名: ${JOB_NAME} 生产 环境发布失败" -C "点击 ${RUN_DISPLAY_URL} 查看详情"'
                                }
                            } 
                        }
                    }
                }
            }
        }

        stage('发布完成') {
            steps {
                sh '${JENKINS_HOME}/ansible/msg.py -n -u "${ops_group},${dev_group},${qa_group},${boss_group}" -T "项目名: ${JOB_NAME} 发布成功" -C "项目名称:  ${JOB_NAME}\n提交信息:  ${GIT_COMMIT_TITLE}\n提交用户:  ${GIT_COMMIT_USER}\n版本HASH:  ${GIT_COMMIT_HASH}\n本次发布完成。"'
            }
        }
    }
}