def requestBody = []
def stepCount = 0
def generalStatus = 'PASS'
def PROJECT_ID = "OBDMARKETS"
def SUBSYSTEM = "C01"
def PLATFORM = "c"
def SOF = "C0PD3"
def JCL_DETAILS = "C0PD3MPP"
def MAINFRAME_PASS = "C01_Machineuser_Mainframe"
def OUTPUT_FILE = "C01TN000.DSNN.C0PD3.OUTTEMP"
def FAIL_FILE = "C01TN000.DSNN.C0PD3.OUTFAIL"
def MAIL = "owen.paesschesoone2@kbc.be"
def BUILD_MAIL= "owen.paesschesoone2@kbc.be"

pipeline {
    agent {
        kubernetes {
            cloud 'Openshift PRO'
            label 'y45-openshift-s'
            defaultContainer 'jnlp'
        }
    }
    tools {
        jdk 'JDK_1.8.0_Latest'
    }
    post {
        always {
            script {
                if (currentBuild.currentResult == 'FAILURE') {
                    step([$class: 'Mailer', notifyEveryUnstableBuild: true, recipients: "$BUILD_MAIL", sendToIndividuals: true])
                }
            }
        }
    }

    stages {
        // Stage-5 Submitting the SOF/MPP & Validation job in mainframe
        stage("Submitting Job") {
            steps {
                script {
                    def req4data = "'$SUBSYSTEM" + "TN000.DSNP.JCL(" + "$JCL_DETAILS" + ")\\'"
                    def req3data = '"file":"//\\'
                    def req5data = '"'
                    def req6data = '{' + "$req3data" + "$req4data" + "$req5data" + '}'
                    def joburl = "https://zosmf${PLATFORM}t.be.srv.dev.sys:32208/zosmf/restjobs/jobs"

                    httpRequest acceptType: 'APPLICATION_JSON',
                            consoleLogResponseBody: true,
                            contentType: 'APPLICATION_JSON',
                            authentication: "$MAINFRAME_PASS",
                            customHeaders: [[maskValue: false, name: 'X-CSRF-ZOSMF-HEADER', value: '""']],
                            httpMode: 'PUT',
                            requestBody: "${req6data}",
                            responseHandle: 'NONE',
                            url: joburl,
                            outputFile: "$SOF" + 'Job.txt',
                            wrapAsMultipart: false
                }
                script
                        {
                            echo 'Waiting 1 min to complete Job'
                            sleep 20 // seconds
                        }
                script {
                    data.each {
                        if (it.contains("JCL_DETAILS")) {
                            JCL_DETAILS = it.split("=")[1]
                        } else if (it.contains("MAINFRAME_PASS")) {
                            MAINFRAME_PASS = it.split("=")[1]
                        } else if (it.contains("SOF")) {
                            SOF = it.split("=")[1]
                        } else if (it.contains("PLATFORM")) {
                            PLATFORM = it.split("=")[1]
                        }
                    }
                    def data1 = readFile(file: "$SOF" + 'Job.txt').split("\"jobid\":\"")[1]
                    def jobid = data1.split("\",\"")[0]
                    def data2 = readFile(file: "$SOF" + 'Job.txt').split("\"jobname\":\"")[1]
                    def jobname = data2.split("\",\"")[0]
                    def url = "https://zosmf${PLATFORM}t.be.srv.dev.sys:32208/zosmf/restjobs/jobs/${jobname}/${jobid}"
                    httpRequest acceptType: 'APPLICATION_JSON',
                            consoleLogResponseBody: true,
                            contentType: 'APPLICATION_JSON',
                            authentication: "$MAINFRAME_PASS",
                            customHeaders: [[maskValue: false, name: 'X-CSRF-ZOSMF-HEADER', value: '""']],
                            responseHandle: 'NONE',
                            url: url,
                            outputFile: "$SOF" + 'Jobout.txt',
                            wrapAsMultipart: false
                }
            }
        }
        // Stage-6 Verify JCL ran successfully or not
        stage("Verifying the JCL Return Code") {
            steps {
                script {
                    def status = ""
                    def record1 = ""
                    def retcode = readFile(file: "$SOF" + 'Jobout.txt').split("\"retcode\":")[1].replace("}", "")
                    if (retcode.contains("CC 0000") || retcode.contains("CC 0004") || retcode.contains("CC 0999") || retcode.contains("null") || retcode.contains("CC 0008")) {
                        status = "PASS"
                        println "JCL submitted successfully"
                        record1 = "JCL submitted successfully"
                    } else {
                        status = "FAIL"
                        println status
                        println "Error code in JCL submission"
                        record1 = "Error code in JCL submission"
                        generalStatus = "FAIL"
                    }
                    println "retcode: $retcode"
                    println "status: $status"
                    requestBody.add(generateStepResult(stepCount,status,record1))
                    stepCount += 1
                    if (status == "FAIL") {
                        error "This pipeline stops here!"
                    }
                }
            }
        }
        // Stage 7 - Reading the Output File generated in Mainframe
        stage("Reading output file generated in Mainframe")
                {
                    steps
                            {
                                script {

                                    def outurl = "https://zosmf${PLATFORM}t.be.srv.dev.sys:32208/zosmf/restfiles/ds/$OUTPUT_FILE"
                                    httpRequest consoleLogResponseBody: true, contentType: 'APPLICATION_JSON',
                                            authentication: "$MAINFRAME_PASS",
                                            customHeaders: [[maskValue: false, name: 'X-CSRF-ZOSMF-HEADER', value: '""']],
                                            httpMode: 'GET',
                                            outputFile: "$SOF" + 'File.txt',
                                            encoding: 'UTF-8',
                                            responseHandle: 'NONE',
                                            url: outurl,
                                            wrapAsMultipart: false
                                }
                            }
                }
        // Stage 8 - Reading the Fail Test Case File generated in Mainframe
        stage("Reading FAIL TEST CASE file generated in Mainframe")
                {
                    steps
                            {
                                script {

                                    failurl = "https://zosmf${PLATFORM}t.be.srv.dev.sys:32208/zosmf/restfiles/ds/$FAIL_FILE"
                                    httpRequest consoleLogResponseBody: true, contentType: 'APPLICATION_JSON',
                                            authentication: "$MAINFRAME_PASS",
                                            customHeaders: [[maskValue: false, name: 'X-CSRF-ZOSMF-HEADER', value: '""']],
                                            httpMode: 'GET',
                                            outputFile: "$SOF" + 'Fail.txt',
                                            responseHandle: 'NONE',
                                            url: failurl,
                                            wrapAsMultipart: false
                                }
                                script {

                                    def retfile = readFile("$SOF" + 'Fail.txt')
                                    if (retfile.size() == 0) {
                                        println "No Failed Test Cases"
                                    } else {
                                        println "Mail Valid Report in case of Failed Test Cases"
                                        emailext attachmentsPattern: "$SOF" + 'File.txt',
                                                body: '''Please review the validation test results ''',
                                                subject: 'FET-' + "$SOF" + '_VALIDATION REPORT',
                                                to: "$MAIL"
                                    }
                                }
                            }
                }
            stage("Parsing output file & return the results to parent") {
                steps {
                    script {

                        def data1 = readFile(file: "$SOF" + 'File.txt').split('\n')
                        String flag = "false"
                        def records = ""
                        def testCaseKey = ""
                        def projectKey = ""
                        def reportObject = ""
                        data1.each {
                            if (it.contains("-----------------------------------------------------") && flag == "false") {
                                flag = "true"
                            } else if (it.contains("TEST-CASE-RESULT :")) {
                                def status = it.split(":")[1].trim()
                                     if (status == "FAIL")
                                        {
                                           generalStatus = "FAIL"
                                        }
                                requestBody.add(generateStepResult(stepCount, status, records))

                                stepCount += 1
                                flag = "false"
                                records = ""
                            } else if (flag == "true") {
                                if (it.contains("TEST-CASE-ID")) {
                                    testCaseKey = it.split(":")[1]
                                    projectKey = testCaseKey.split("-")[0]
                                    println "ProjectKey: ${projectKey} -- TestCaseKey: ${testCaseKey}"
                                } else {
                                    records = records + it + " <br/>"
                                }
                            }
                        }
                        reportObject = (generateReportData(requestBody, generalStatus, testCaseKey))

                        writeFile file: "report.json", text: reportObject

                        archiveArtifacts artifacts: "report.json", fingerprint: true
                    }
            }
        }
    }
}
//Common function to create testcase list
def generateStepResult(int stepCount, String status, String comment) {

    def data = """{
        "index" : "${stepCount}",
        "status": "${status}",
        "comment": "${comment}"
    }"""
    return data
}

//Common function to create testcase list
def generateReportData(List requestBody, String status, String testCaseKey) {
    def data = """{
        "testCaseKey" : "${testCaseKey}",
        "status": "${status}",
        "environment": "FET",
        "executionTime": 180000,
        "comment": "The test results has been updated via Jenkins job",
        "scriptResults": ${requestBody}
    }"""
    return data
}
