def fullname            = "${env.BUILD_NUMBER}"

// SERVICE
def dependecy_tools     = "${env.DEPENDENCY}"
def service_code_type   = "${env.SERVICE_CODE_TYPE}"
def approval            = []
def thresholdApproval   = []
def reviewer            = []
def tresholdReviewer    = []

// URL
def secret_url          = ""
def docker_server       = ""
def gitops_server_dc    = ""
def gitops_server_drc   = ""
def sast_url            = ""
def approvalReviewerUrl = ""

// NOTIFICATION
def job_success         = "SUCCESS"
def job_error           = "ERROR"

// Credentials
def docker_credentials  = "nexus_cred"
def repo_credentials    = "gitlab_cred"

//Proxy 
def http_proxy          = ""
def https_proxy         = ""

//Git Account
def git_email           = ""
def git_username        = ""

//Log Function 
def log_node            = ""
def log_location        = ""

//Variable Initialization
def runPipeline, runBranch, scmCode, scmDockerfile, scmDependency, mergeSourceBranch
def pullCode, version, projectId, repoBranch, repoURL, sast_scan_url, mergeStatus, mergeIdCheck, manualMerge
def latestImageId, latestImageArgoId, syncStatus, healthStatus, quality_gate_name
def docker_url, pipeline, dockerPushed, applicationName, jenkinsPath, repo_url, repo_dockerfile_url, testing_url
def currentTagStaging, newTagStaging, testingEnabled, repo_dependency_url, gitopsServer, applicationList
def app_name, service_type, service_name, get_trigger, googlechat_url, applicationNameDRC, applicationNameDC

//Log Function Initialization
def sonarScannerSummary = []
def trivyScannerSummary = []
def unitTestResult, securityTestResult, automationTestResult, perfTestResult

node ('slave1') {
    environment {
    GIT_USER_EMAIL = "${git_email}"
    GIT_USER_NAME = "${git_username}"
    }
    try{       
        stage ("Checkout") {
            // Clean Workspace
            cleanWs()

            // Get Value from Jenkins pipeline
            jenkinsPath = env.JOB_NAME
            echo "Jenkins Path : ${jenkinsPath}"

            def testPath = jenkinsPath.split("\\/")

            get_trigger    = testPath[0].toString()
            app_name       = testPath[1].toString()
            service_type   = testPath[2].toString()
            service_name   = testPath[3].toString()

            echo "Environment: $get_trigger"
            echo "App: $app_name"
            echo "Service Type: $service_type"
            echo "Service Name: $service_name"
            
            //Retrieve Reviewer and Approval data
            if( get_trigger != "develop"){
                echo "approvalReviewerUrl :${approvalReviewerUrl}"
                echo "get_trigger :${get_trigger}"
                echo "app_name :${app_name}"
                
                approval            = listApprovalReviewer(url: approvalReviewerUrl, action: "workflow-approval", environment: get_trigger, application: app_name.toUpperCase())
                thresholdApproval   = listThresholdApprovalReviewer(url: approvalReviewerUrl, action: "workflow-approval", environment: get_trigger, application: app_name.toUpperCase())
                // Show Result Approval
                echo "List for Approval"
                for (subArrayApproval in approval) {
                    echo "Sub-array Approval: ${subArrayApproval}"
                }

                echo "List for Treshold"
                for (subArrayTresholdApproval in thresholdApproval) {
                    echo "Sub-array Threshold Approval: ${subArrayTresholdApproval}"
                }
                
                if ( get_trigger == "production" ){
                    reviewer            = listApprovalReviewer(url: approvalReviewerUrl, action: "workflow-review", environment: get_trigger, application: app_name.toUpperCase())
                    tresholdReviewer    = listThresholdApprovalReviewer(url: approvalReviewerUrl, action: "workflow-review", environment: get_trigger, application: app_name.toUpperCase())
                    // Show Result Treshold
                    for (subArrayReviewer in reviewer) {
                        echo "Sub-array Reviewer: ${subArrayReviewer}"
                    }
                    for (subArrayTresholdReviewer in tresholdReviewer) {
                        echo "Sub-array Treshold Reviewer: ${subArrayTresholdReviewer}"
                    }
                }
            }
            
            // Assign Value from Jenkins
            if ( service_code_type == "null" ){
                service_code_type = "Test"
            }
            echo "Service Code Type : $service_code_type"
            repo_url            = ""
            repo_dockerfile_url = ""
            testing_url         = ""
            credentials_url     = ""
            repo_dependency_url = ""
            docker_url          = "${docker_server}/${get_trigger}/${app_name}/${service_type}/${service_name}"
            version             = "${get_trigger}"
            sast_scan_url       = "${sast_url}/dashboard?id=${app_name}-${service_name}"
            quality_gate_name   = "${app_name}"
            projectId           = getProjectId(app_name: app_name, service_type: service_type, service_name: service_name)

            // Application Name for Gitops
            applicationNameDRC  = "${service_type}-${service_name}-${get_trigger}-drc"
            applicationNameDC   = "${service_type}-${service_name}-${get_trigger}-dc"

            // Get Credentials for Google Chat Notification
            withCredentials([string(credentialsId: "googlechat-${app_name}", variable: 'GOOGLE_CHAT_WEBHOOK')]) {
                googlechat_url = "${GOOGLE_CHAT_WEBHOOK}"
            }  

            // Assign Value for deployment checking
            echo "Start Assign Value Deployment Checking"
            if (get_trigger == "production"){
                gitopsServer = [gitops_server_drc]
                applicationList = [applicationNameDRC]
            } else {
                gitopsServer = [gitops_server_drc]
                applicationList = [applicationNameDRC]
            }

            // TRIGGER
            if (get_trigger == 'test') {
                runPipeline = 'test'
                runBranch   = "*/release"
                repoBranch  = "release"
                pullCode    = "develop"
            } else if (get_trigger == 'production') {
                runPipeline = 'production'
                runBranch   = "*/master"
                repoBranch  = "master"
                pullCode    = "release"
            } else if (get_trigger == 'staging') {
                runPipeline = 'staging'
                runBranch   = "*/staging"
                repoBranch  = "staging"
                pullCode    = "release"
            } else if (get_trigger == 'develop'){
                runPipeline = 'develop'
                runBranch   = "*/develop"
                repoBranch  = "develop"
                pullCode    = "develop"
            }else {
                runPipeline = 'dev'
                runBranch   = "*/${env.BRANCH_NAME}"
            }

            echo "With branch ${runBranch}"

            // Create directory for requirement code
            echo "Start create directory for code and dependency"
            if (fileExists("code") != true){
                sh "mkdir code"
            } else {
                sh "rm -r code"
                sh "mkdir code"
            }

            if (fileExists("dockerfile") != true){
                sh "mkdir dockerfile"
            }

            if(dependecy_tools == "true"){
                if (fileExists("dependency") != true){
                    sh "mkdir dependency"
                } else {
                    sh "rm -r dependency"
                    sh "mkdir dependency"
                }
            }

            if (fileExists("secret") != true){
                sh "mkdir secret"
            }
            
            if (fileExists("testDir") != true){
                sh "mkdir testDir"
            }
        
            // SOURCE CHECKOUT
            echo "Start checkout for code and dependency"

            mergeSourceBranch = "${env.gitlabSourceBranch}"

            if ( runPipeline == 'test' || runPipeline == 'staging' || runPipeline == 'production'){
                dir("code"){
                    scmCode = checkout([
                    $class: 'GitSCM', 
                    branches: [[name: "origin/${env.gitlabSourceBranch}"]], 
                    userRemoteConfigs: [[
                        credentialsId: repo_credentials, 
                        url: repo_url
                    ]]])

                    mergeSourceBranch = "${env.gitlabSourceBranch}"
                    echo "Merge Source Branch : ${mergeSourceBranch}"
                }
            } else if( runPipeline == 'develop'){
                dir("code"){
                    scmCode = checkout([
                    $class: 'GitSCM', 
                    branches: [[name: 'develop']], 
                    userRemoteConfigs: [[
                        credentialsId: repo_credentials, 
                        url: repo_url
                    ]]])
                }
            } else {
                dir (code){
                    "No Pull Code"
                }
            }
            
            // Dockerfile CHECKOUT
            dir("dockerfile"){
                scmDockerfile = checkout([
                $class: 'GitSCM', 
                branches: [[name: "*/master"]], 
                userRemoteConfigs: [[
                    credentialsId: repo_credentials, 
                    url: repo_dockerfile_url
                ]]])
            }

            // Secret CHECKOUT
            dir("secret"){
                scmDockerfile = checkout([
                $class: 'GitSCM', 
                branches: [[name: "*/master"]], 
                userRemoteConfigs: [[
                    credentialsId: repo_credentials, 
                    url: credentials_url
                ]]])
            
                //Copy env and configurations files and folder to source code
                def configFolder = "${app_name}/${service_type}/${service_name}"
                def envFolder    = "${configFolder}/env"
                def folderExist = sh(script: "[ -d '${envFolder}' ] && echo 'Exist' || echo 'Not Exist'", returnStdout: true).trim()
                def itemCount = sh(script: "ls -A ${envFolder} | wc -l", returnStdout: true).trim().toInteger()
                def isNotEmpty = sh(script: "[ \$(ls -A ${envFolder} | wc -l) -gt 0 ]", returnStatus: true) == 0

                if (folderExist == "Exist"){
                    if (isNotEmpty){
                        echo "Item Count = ${itemCount}"
                        def testExist= fileExists("${envFolder}/.env")
                        echo "File Exist : ${testExist}"
                        if (fileExists("${envFolder}/.env") && itemCount == 1) {
                            echo "Hanya ada .env files"
                            sh "cp ${configFolder}/env/.env ../code"
                        } else {
                            echo "Ada beberapa file selain .env"
                            sh "cp -r ${configFolder}/env/* ../code"
                            if (fileExists("${configFolder}/env/.env") == true){
                                sh "cp ${configFolder}/env/.env ../code"
                            }
                        }
                    } else {
                        echo "Env Folder is exist but empty"
                    }
                } else {
                    echo " There is no env folder"
                }                      
            }

            // Testing Checkout 
            dir("testDir"){
                testCode = checkout([
                $class: 'GitSCM', 
                branches: [[name: "master"]], 
                userRemoteConfigs: [[
                    credentialsId: repo_credentials, 
                    url: testing_url
                ]]])

                //Copy env to source code
                if (fileExists("${service_type}/${service_name}.groovy") == true){
                    pipeline = load "${service_type}/${service_name}.groovy"
                    testingEnabled = true
                } else {
                    echo "no testing script yet"
                    testingEnabled = false
                }
            }

            // Dependency CHECKOUT
            if(dependecy_tools == "true"){
                dir("dependency"){
                    scmDependency = checkout([
                    $class: 'GitSCM', 
                    branches: [[name: "*/master"]], 
                    userRemoteConfigs: [[
                        credentialsId: repo_credentials, 
                        url: repo_dependency_url
                    ]]])

                    sh "cp -r * ../code"
                }     
            }

            echo "-----------------------LIST FILE IN CODE-----------------------------"
            sh "ls -al code/"   

            // Tag Versioning
            if ( version == "staging" || version == "production"){
                dir("code"){
                    withCredentials([usernamePassword(credentialsId: 'gitlab_cred', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                        def encodedUsername = URLEncoder.encode(GIT_USERNAME, "UTF-8")
                        def encodedPassword = URLEncoder.encode(GIT_PASSWORD, "UTF-8")

                        def tag_repository  = "https://${encodedUsername}:${encodedPassword}@git_url/${app_name}/${service_type}/${service_name}.git"
                        currentTagStaging = getCurrentTag(version: "staging", repo: tag_repository)
                        currentTagProduction = getCurrentTag(version: "production", repo: tag_repository)
                        echo "Current Tag Staging: ${currentTagStaging}"
                        echo "Current Tag Production: ${currentTagProduction}"
                        
                        newTagStaging = incrementVersion(version:"staging", currentTagStaging: currentTagStaging, currentTagProduction: currentTagProduction)
                        echo "New Tag Staging: ${newTagStaging}"
                        newTagProduction = incrementVersion(version:"production", currentTagStaging: currentTagStaging, currentTagProduction: currentTagProduction)
                        echo "New Tag Production: ${newTagProduction}"
                    }    
                }
            }
        }

        if (version != 'develop') {
            stage ("Merge Conflict Check") {
                echo "Start Merge Conflict Check"

                withCredentials([usernamePassword(credentialsId: 'gitlab_cred', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                    def encodedUsername = URLEncoder.encode(GIT_USERNAME, "UTF-8")
                    def encodedPassword = URLEncoder.encode(GIT_PASSWORD, "UTF-8")

                    sh """
                    git config --global http.proxy ${http_proxy}
                    git config --global https.proxy ${http_proxy}
                    git config --global user.email ""
                    git config --global user.name ""
                    """

                    dir("code") {
                        // Delete local tags
                        echo "Merge Check"
                        sh "git remote set-url origin https://${encodedUsername}:${encodedPassword}@git_url/${app_name}/${service_type}/${service_name}.git"
                        sh "git fetch origin ${repoBranch}"
                        def mergeOutput = sh(script: "git merge-base HEAD origin/${repoBranch}", returnStdout: true).trim()
                        if (mergeOutput.startsWith('CONFLICT')) {
                            error "Merge conflict detected. Resolve conflicts before merging to the release branch."
                            currentBuild.result = 'ABORTED'
                        } else {
                            echo "No merge conflicts detected. It's safe to merge to the release branch."
                        }
                    }
                }
            }
        }
    
        // // DEV-STAGING
        if (version != "production" && version != "staging") {

            // Unit Test
            stage ("Unit Test") {
                echo "Start Unit Testing"
                withEnv(['PATH=/usr/local/go/bin:' + env.PATH]) {
                    if ( testingEnabled == true ){
                        pipeline.unitTest()
                    } else {
                        echo "No Unit Test"
                    }
                }
            }
            
            //Sonarqube Analysis
            if (service_code_type == "maven"){
                stage ("Build Jar File") {
                    echo "Build Jar File"
                    dir("code"){
                        sh "/opt/apache-maven-3.9.5/bin/mvn --settings /opt/settings.xml clean install"
                    }
                }
            }

            stage ("SAST & SCA Code") {
                echo "Start SAST"
                withSonarQubeEnv('sonar-scanner'){
                    dir("code"){
                        if (version == 'test'){
                            sonarScanner(app_name: "${app_name}-${service_name}", quality_gate_name: quality_gate_name, service_name: "${app_name}_${service_name}", version: version, sonarSrc: ".", sonarBranch: repoBranch, quality_gate:"true", service_code: service_code_type)
                        } else{
                            sonarScanner(app_name: "${app_name}-${service_name}", quality_gate_name: quality_gate_name, service_name: "${app_name}_${service_name}", version: version, sonarSrc: ".", sonarBranch: repoBranch, quality_gate:"false", service_code: service_code_type)
                        }
                    }
                }
            }

            // Merge Request Approval and Review Checking
            if (version == 'test') {
                stage("MR Approval") {
                    echo "Start MR Approval"
                    timeout(time: 2880, unit: 'MINUTES') {
                        def isApproved = false
                        while (!isApproved) {
                            projectId = getProjectId(app_name: app_name, service_type: service_type, service_name: service_name)
                            echo "Project ID: ${projectId}"
                            //Check Merge Status First
                            echo "Start Manual Merge Check"
                            mergeStatus = getLastMergedStatus(project_id: projectId)
                            if (mergeStatus == "merged"){
                                echo "Merge Request already Merged"
                                mergeIdCheck = getLastMergedId(project_id: projectId)
                                mergedComment(project_id: projectId, mergeRequestId: mergeIdCheck)
                                manualMerge = true
                                error("Build failed due to a manual merge.")
                            } else {
                                echo "Not Merged Yet"
                            }

                            isApproved = isMergeRequestApproved(project_id: projectId, source_branch: mergeSourceBranch, approvers: approval, treshold: thresholdApproval)
                            if (isApproved) {
                                echo "Merge request is approved. Proceeding to Deployment stage."
                                break
                            } else {
                                echo "Merge request is not approved. Pipeline will wait."
                            }
                            sleep(time: 15, unit: 'SECONDS')
                        }
                    }
                }

                stage ("Backup Container"){
                    echo "Start Backup Images"
                    try{
                        docker.withRegistry("https://${docker_server}", docker_credentials) {
                            withCredentials([string(credentialsId: 'gitops-token', variable: 'gitopsCred')]) {
                                dockerBackup(image_name: docker_url, image_version: "test")
                            }
                        }
                    } catch(error){
                        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                            echo "First Time build, no backup yet"
                            sh "exit 1"
                        }
                    }
                }
            }

            //Build Container
            stage ("Build Container") {
                echo "Start Build Docker Images"
                dir("code"){
                    echo "Build Image with Version"
                    dockerBuild(image_name: docker_url, image_version: version, service_type: service_type, service_name: service_name, service_code_type:service_code_type)
                }
            }

            stage ("Container Security Testing") {
                echo "Start Trivy checking Docker Images"
                dir("code"){
                    sh "https_proxy=${http_proxy} trivy image --scanners vuln ${docker_url}:${version}"
                }
            }
            
            stage ("Store Binary to Artifactory") {
                echo "Start Push Docker Images"
                dir("code"){
                    docker.withRegistry("https://${docker_server}", docker_credentials) {
                        echo "Push Docker Images"
                        dockerPush(image_name: docker_url, image_version: version)
                        latestImageId = digestId(image_name: "${docker_url}")
                        echo "Latest Image ID : ${latestImageId}"
                        dockerRemove(image_name: docker_url, image_version: version)
                        dockerPushed = true
                    }
                }
            }
        } else {
            // Production and Staging Deployment
            // Document Review for production deployment
            if ( version == 'production'){
                stage("MR Review") {
                    echo "Start Review"
                    timeout(time: 2880, unit: 'MINUTES') {
                        def isReviewed = false
                        while (!isReviewed) {
                            projectId = getProjectId(app_name: app_name, service_type: service_type, service_name: service_name)
                            echo "Project ID: ${projectId}"
                            isReviewed = isMergeRequestReviewed(project_id: projectId, source_branch: mergeSourceBranch, reviews: reviewer, treshold: tresholdReviewer)
                            if (isReviewed) {
                                echo "Merge request is Reviewed. Proceeding to Merge Request Approval."
                                break
                            } else {
                                echo "Merge request is not Reviewed. Pipeline will wait."
                            }
                            sleep(time: 15, unit: 'SECONDS')
                        }
                    }
                }
            }

            stage ("MR Approval") {
                echo "Start MR Check"
                timeout(time: 2880, unit: 'MINUTES') {
                    def isApproved = false
                    while (!isApproved) {
                        projectId = getProjectId(app_name: app_name, service_type: service_type, service_name: service_name)
                        echo "Project ID: ${projectId}"
                        //Check Merge Status First
                        echo "Start Manual Merge Check"
                        mergeStatus = getLastMergedStatus(project_id: projectId)
                        if (mergeStatus == "merged"){
                            echo "Merge Request already Merged"
                            mergeIdCheck = getLastMergedId(project_id: projectId)
                            mergedComment(project_id: projectId, mergeRequestId: mergeIdCheck)
                            manualMerge = true
                            error("Build failed due to a manual merge.")
                        } else {
                            echo "Not Merged Yet"
                        }

                        isApproved = isMergeRequestApproved(project_id: projectId, source_branch: mergeSourceBranch, approvers: approval, treshold: thresholdApproval)
                        if (isApproved) {
                            echo "Merge request is approved. Proceeding to Deployment stage."
                            break
                        } else {
                            echo "Merge request is not approved. Pipeline will wait."
                        }
                        sleep(time: 15, unit: 'SECONDS')
                    }
                }
            }  
            
            stage ("Backup Container"){
                echo "Start Backup Images"
                try{
                    docker.withRegistry("https://${docker_server}", docker_credentials) {
                        withCredentials([string(credentialsId: 'gitops-token', variable: 'gitopsCred')]) {
                            def tagDeployImage = backupImageTag(application: applicationNameDRC, version: version, token: gitopsCred, server: gitops_server_drc)
                            dockerBackup(image_name: docker_url, image_version: tagDeployImage)
                        }
                    }
                } catch(error){
                    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                        echo "First Time build, no backup yet"
                        sh "exit 1"
                    }
                }
            }

            stage ( "Update Container" ){
                docker.withRegistry("https://${docker_server}", docker_credentials) {
                    if ( version == 'production' ){
                        latestImageId = "${docker_url}:${newTagProduction}"
                        echo "Latest Image ID : ${latestImageId}"
                        dockerUpdateImage(version: version, app_name: app_name, service_type: service_type, service_name: service_name, sourceTag: "${currentTagStaging}", targetTag: "${newTagProduction}")
                    } else {
                        latestImageId = "${docker_url}:${newTagStaging}"
                        echo "Latest Image ID : ${latestImageId}"
                        dockerUpdateImage(version: version, app_name: app_name, service_type: service_type, service_name: service_name, sourceTag: "test", targetTag: "${newTagStaging}")
                    }
                    dockerPushed = true
                }
            }
        }

        //Deployment Checking
        stage ("Deployment Checking"){
            echo "Start Deployment Checking"
            try{
                timeout (time: 10, unit: 'MINUTES') {
                    for (int i = 0; i < gitopsServer.size(); i++) {
                        withCredentials([string(credentialsId: 'gitops-token', variable: 'gitopsCred')]) {
                            sleep(time: 10, unit: "SECONDS")
                            latestImageArgoId = getImageId(application: applicationList[i], version: version, token: gitopsCred, server: gitopsServer[i])
                            echo "latestImageArgoId : ${latestImageArgoId}"
                            echo "latestImageId : ${latestImageId}"
                            if (latestImageArgoId != latestImageId){
                                defImageStatus = "different"
                                while(defImageStatus == "different"){
                                    syncStatus = getSyncStatus(application: applicationList[i], token: gitopsCred, server: gitopsServer[i]) 
                                    echo "syncStatus : ${syncStatus}"

                                    latestImageArgoId = getImageId(application: applicationList[i], version: version, token: gitopsCred, server: gitopsServer[i])
                                    echo "latestImageArgoId : ${latestImageArgoId}"
                                    echo "latestImageId : ${latestImageId}"
                                    if (latestImageArgoId != latestImageId){
                                        defImageStatus = "different"
                                    } else{
                                        defImageStatus = "same"
                                    }
                                    sleep(time: 10, unit: "SECONDS")
                                }
                            } else {
                                healthStatus = getHealthStatus(application: applicationList[i], token: gitopsCred, server: gitopsServer[i])
                                echo "Healthy Status : ${healthStatus}"
                                if (healthStatus != "Healthy"){
                                    defHealthStatus = "notHealthy"
                                    while(defHealthStatus == "notHealthy"){
                                        healthStatus = getHealthStatus(application: applicationList[i], token: gitopsCred, server: gitopsServer[i])
                                        if (defHealthStatus != "Healthy"){
                                            defHealthStatus = "notHealthy"
                                        } else{
                                            defHealthStatus = "Healthy"
                                        }
                                    }
                                } else {
                                    echo "health status is healthy"
                                }
                            }
                        }
                    }
                }
            } catch (error) {
                def user = error.getCauses()[0].getUser()
                echo "User : ${user}"
                if('SYSTEM' == user.toString()) { 
                    echo "Timeout reached, continue to next stage"
                } 
                else {
                    throw error
                }
            }
        }
        
        // Testing Flow After Deployment
        if (version == 'test' || version == 'staging'){
            if ( version == 'test'){
                stage('Run Tests') {
                    parallel (
                        'Security': {
                            stage ("BurpSuite Scan") {
                                echo "Starting BurpSuite Scan"
                                if ( testingEnabled == true ){
                                    pipeline.burpsuiteScan()
                                } else {
                                    echo "No Security Test"
                                }
                            }
                        },
                        'Automation': {
                            stage ("Automation Testing") {
                                echo "Start Automations Testing"
                                if ( testingEnabled == true ){
                                    pipeline.automationTest()
                                } else {
                                    echo "No Security Test"
                                }
                            }
                        }
                    )
                }
            } else {
                stage ("Performance Testing") {
                    echo "Start Performance Testing"
                    if ( testingEnabled == true ){
                        pipeline.performanceTest()
                    } else {
                        echo "No Security Test"
                    }
                }
            }
        } 



        // Autotagging
        if (version == 'production' || version == 'staging') {
            stage ("Auto Tagging") {
                echo "Tagging "
                def newTag
                if ( version == 'staging'){
                    newTag = newTagStaging
                } else {
                    newTag = newTagProduction
                }
                echo "Tagging git with ${newTag}"

                withCredentials([usernamePassword(credentialsId: 'gitlab_cred', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                    def encodedUsername = URLEncoder.encode(GIT_USERNAME, "UTF-8")
                    def encodedPassword = URLEncoder.encode(GIT_PASSWORD, "UTF-8")

                    sh """
                    git config --global http.proxy ${http_proxy}
                    git config --global https.proxy ${http_proxy}
                    git config --global user.email ""
                    git config --global user.name ""
                    """
                    dir("code") {
                        // Delete local tags
                        sh "git tag | xargs git tag -d"
                        // Checking the tag using echo and grep
                        echo "Checking tag for: ${newTag}"
                        while (sh(script: 'git tag | grep \'^' + newTag + '\$\'', returnStatus: true) == 0) {
                            echo "Tag ${newTag} already exists!"
                            newTag = incrementVersion(newTag)
                            echo "Trying new tag: ${newTag}"
                        }                                
                        sh "git tag ${newTag}"
                        sh "git remote set-url origin https://${encodedUsername}:${encodedPassword}@git_url/${app_name}/${service_type}/${service_name}.git"
                        sh "git push origin refs/tags/${newTag}"
                    }
                }

                echo "Generated release notes for ${newTag}"
                // def releaseNotes = generateReleaseNotes(currentTag)
                // writeFile(file: "RELEASE_NOTES.md", text: releaseNotes)
                // Set the DOCKER_VERSION_TAG
                env.DOCKER_VERSION_TAG = newTag
            }
        }
        
        // Accept Merge Request
        if (version != 'develop') {
            stage ("Accept Merge Request") {
                echo "Start Merge Request"
                def mergeRequestId = getLatestMergeRequestId(project_id: projectId, source_repo: mergeSourceBranch)
                echo "Merge Request ID : ${mergeRequestId}"
                if (mergeRequestId != null) {
                    echo "Accepting merge request ID ${mergeRequestId}"
                    acceptMergeRequest(project_id: projectId, mergeRequestId: mergeRequestId)
                } else {
                    echo "No merge request ID found"
                }
            }
        }
 
        // Notification
        stage ("Notification"){
            script {
                echo "Notification"
                def webhookUrl = "${googlechat_url}"
                def messageSend = "{\"cardsV2\": [{\"cardId\": \"createCardMessage\",\"card\": {\"header\": {\"title\": \"Application has been successfully deployed on ${version} environment\",\"subtitle\": \"${app_name} - ${service_type} - ${service_name}\"},\"sections\": [{\"widgets\": [{\"buttonList\": {\"buttons\": [{\"text\": \"Go to the deployment\",\"onClick\": {\"openLink\": {\"url\": \"${env.BUILD_URL}\"}}}]}}]}]}}]}"
                
                sh(script: """
                    curl --http1.1 -x ${http_proxy} \
                    -H 'Content-Type: application/json; charset=UTF-8' \
                    -d '${messageSend}' \
                    "${webhookUrl}"
                """)

                // Delete Code, Secret and Dependency
                sh 'docker builder prune -f && rm -rf code dependency secret'
            }
        }

    } catch (e) {
        echo "Job Failed"
        if (dockerPushed == true) {
            if (version != 'develop') {
                def newTag
                if (version == 'staging') {
                    newTag = incrementVersionRevert(version:version, currentTagStaging: newTagStaging, currentTagProduction: newTagProduction)
                } else if ( version == 'production'){
                    newTag = incrementVersionRevert(version:version, currentTagStaging: newTagStaging, currentTagProduction: newTagProduction)
                } else {
                    newTag = "test"
                }

                stage("Revert Container") {
                    echo "Revert Container ${version.capitalize()}"
                    echo "Value of newTag: ${newTag}"
                    def image_name = "${docker_server}/${version}/${app_name}/${service_type}/${service_name}"

                    try{
                        docker.withRegistry("https://${docker_server}", docker_credentials) {
                            dockerPull(image_name: "${image_name}", image_version: "revert")
                            sh "docker tag ${image_name}:revert ${image_name}:${newTag}"
                            dockerPush(image_name: docker_url, image_version: newTag)
                            dockerRemove(image_name: docker_url, image_version: newTag)
                            dockerRemove(image_name: "${image_name}", image_version: "revert")
                        }
                    } catch(error){
                        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                            echo "First Time build, no backup yet"
                            sh "exit 1"
                        }  
                    }
                }
                
                if ( version != 'test'){
                    stage ("Autotagging"){
                        withCredentials([usernamePassword(credentialsId: 'gitlab_cred', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                            def encodedUsername = URLEncoder.encode(GIT_USERNAME, "UTF-8")
                            def encodedPassword = URLEncoder.encode(GIT_PASSWORD, "UTF-8")

                            sh """
                            git config --global http.proxy ${http_proxy}
                            git config --global https.proxy ${http_proxy}
                            git config --global user.email ""
                            git config --global user.name ""
                            """

                            dir("code"){
                                // Delete local tags
                                sh "git tag | xargs git tag -d"
                                sh "git tag ${newTag}"
                                sh "git remote set-url origin https://${encodedUsername}:${encodedPassword}@git_url/${app_name}/${service_type}/${service_name}.git"
                                sh "git push origin refs/tags/${newTag}"
                            }
                        }
                    } 
                }
            }
        } 

        //Comment and Autoclose merge request
        if ( version != "develop" && manualMerge != true){
            stage ("Comment and Close Merge Request") {
                echo "Start Comment Close Merge Request"
                projectId = getProjectId(app_name: app_name, service_type: service_type, service_name: service_name)
                echo "Project ID: ${projectId}"
                def mergeRequestId
                echo "Merge ID: ${mergeRequestId}"
                mergeRequestId = getLatestMergeRequestId(project_id: projectId, source_repo: mergeSourceBranch)
                echo "Line 198 - Merge Request ID : ${mergeRequestId}"
                if (mergeRequestId != null) {
                    echo "Comment and close merge request ID ${mergeRequestId}"
                    noteCloseMergeRequest(project_id: projectId, mergeRequestId: mergeRequestId)
                    closeMergeRequest(project_id: projectId, mergeRequestId: mergeRequestId)
                } else {
                    echo "No merge request ID found"
                }
            }
        }

        // Notification
        stage ("Notification"){
            script {
                echo "Notification"
                def webhookUrl = "${googlechat_url}"
                def messageSend = "{\"cardsV2\": [{\"cardId\": \"createCardMessage\",\"card\": {\"header\": {\"title\": \"Application failed to deploy on ${version} environment\",\"subtitle\": \"${app_name} - ${service_type} - ${service_name}\"},\"sections\": [{\"widgets\": [{\"buttonList\": {\"buttons\": [{\"text\": \"Go to the deployment\",\"onClick\": {\"openLink\": {\"url\": \"${env.BUILD_URL}\"}}}]}}]}]}}]}"
                
                sh(script: """
                    curl --http1.1 -x ${http_proxy} \
                    -H 'Content-Type: application/json; charset=UTF-8' \
                    -d '${messageSend}' \
                    "${webhookUrl}"
                """)

                // Delete Code, Secret and Dependency
                sh 'docker builder prune -f && rm -rf code dependency secret'
            }
        }
        currentBuild.result = "FAILED"
    }

}  

//Auto Tagging function
def getCurrentTag(Map args) {
    try {
        sh "git tag | xargs git tag -d"
        def version = "${args.version}"
        def repo = "${args.repo}"
         if ( version == "staging" ){
            sh "git ls-remote --tags ${repo}"
            return sh(script: """git ls-remote --tags ${repo} | grep -Eo "v[0-9]+\\.[0-9]+\\.[0-9]+\\-rc\\.[0-9]+" | sort -V | tail -n 1""", returnStdout: true).trim()
        } else {
            sh "git ls-remote --tags ${repo}"
            return sh(script: """git ls-remote --tags ${repo} | grep -Eo "v[0-9]+\\.[0-9]+\\.[0-9]+\$" | sort -V | tail -n 1""", returnStdout: true).trim()
        }
        
        
    } catch (Exception e) {
        echo "No tags found. Returning default."
        return '0.0.0'
    }
}

def compareVersions(String v1, String v2) {
    def v1Parts = v1.tokenize('.-rc')
    def v2Parts = v2.tokenize('.-rc')

    for (int i = 0; i < Math.min(v1Parts.size(), v2Parts.size()); i++) {
        def part1 = v1Parts[i]
        def part2 = v2Parts[i]

        if (part1.isInteger() && part2.isInteger()) {
            int intPart1 = part1.toInteger()
            int intPart2 = part2.toInteger()

            if (intPart1 > intPart2) {
                return 1
            } else if (intPart1 < intPart2) {
                return -1
            }
        } else {
            int compareResult = part1.compareTo(part2)
            if (compareResult != 0) {
                return compareResult
            }
        }
    }

    if (v1Parts.size() > v2Parts.size()) {
        return 1
    } else if (v1Parts.size() < v2Parts.size()) {
        return -1
    }

    return 0
}


def incrementVersion(Map args) {
    def currentStaging = "${args.currentTagStaging}"
    def currentProduction = "${args.currentTagProduction}"
    def version= "${args.version}"
    echo "Version: ${version}"

    def comparisonResult = compareVersions(currentStaging, currentProduction)
    echo "Comparison result: ${comparisonResult}"

    def stagingParts   = currentStaging.replaceAll("v", "").replaceAll("-rc.*", "").split("\\.")
    def stagingPartsRc = currentStaging.replaceAll("v", "").replaceAll("-rc", "").split("\\.")
    echo "stagingPart : ${stagingPartsRc}"
    def productionParts = currentProduction.replaceAll("v", "").split("\\.")
    echo "productionParts : ${productionParts}"

    def branchTrigger = "${env.gitlabSourceBranch}"
    echo "branchTrigger : ${branchTrigger}"
    
    if ( version == "staging"){
        def lastCommitMessage = sh(script: 'git log -1 --pretty=%B', returnStdout: true).trim()

        if ( stagingParts == productionParts | comparisonResult < 0) {
            def major = productionParts[0].toInteger()
            def minor = productionParts[1].toInteger()
            def patch = productionParts[2].toInteger()
            def rc    = 0
            if (lastCommitMessage.contains("[MAJOR]")) {
                major++
                minor = 0
                patch = 0
                rc    = 1
            } else if (lastCommitMessage.contains("[MINOR]")) {
                minor++
                patch = 0
                rc    = 1 
            } else {
                echo "673"
                patch = patch + 1
                rc    = 1 
            }

            echo "version: v${major}.${minor}.${patch}-rc.${rc}"
            return "v${major}.${minor}.${patch}-rc.${rc}"

        } else {
            def major = stagingPartsRc[0].toInteger()
            def minor = stagingPartsRc[1].toInteger()
            def patch = stagingPartsRc[2].toInteger()
            def rc    = stagingPartsRc[3].toInteger()

            if (lastCommitMessage.contains("[MAJOR]")) {
            // major++
            // minor = 0
            // patch = 0
            rc++
            } else if (lastCommitMessage.contains("[MINOR]")) {
                // minor++
                // patch = 0
                rc++
            } else {
                // patch = patch + 1
                rc++ 
            }

            return "v${major}.${minor}.${patch}-rc.${rc}"
        }

    } else if ( version == "production"){
        if ( stagingParts == productionParts | comparisonResult < 0) {
            def major = productionParts[0].toInteger()
            def minor = productionParts[1].toInteger()
            def patch = productionParts[2].toInteger()
            patch = patch + 1

            echo "version: v${major}.${minor}.${patch}"
            return "v${major}.${minor}.${patch}"

        } else {
            def major = stagingParts[0].toInteger()
            def minor = stagingParts[1].toInteger()
            def patch = stagingParts[2].toInteger()

            echo "version: v${major}.${minor}.${patch}"
            return "v${major}.${minor}.${patch}"
        }
    }
}

def incrementVersionRevert(Map args) {
    def currentStaging = "${args.currentTagStaging}"
    def currentProduction = "${args.currentTagProduction}"
    def version= "${args.version}"
    echo "Version: ${version}"

    def stagingParts   = currentStaging.replaceAll("v", "").replaceAll("-rc.*", "").split("\\.")
    def stagingPartsRc = currentStaging.replaceAll("v", "").replaceAll("-rc", "").split("\\.")
    echo "stagingPart : ${stagingPartsRc}"
    def productionParts = currentProduction.replaceAll("v", "").split("\\.")
    echo "productionParts : ${productionParts}"
    
    if ( version == "staging"){
        def major = stagingPartsRc[0].toInteger()
        def minor = stagingPartsRc[1].toInteger()
        def patch = stagingPartsRc[2].toInteger()
        def rc    = stagingPartsRc[3].toInteger()

        rc++ 

        return "v${major}.${minor}.${patch}-rc.${rc}"

    } else if ( version == "production"){
        def major = stagingParts[0].toInteger()
        def minor = stagingParts[1].toInteger()
        def patch = stagingParts[2].toInteger()
        patch++

        return "v${major}.${minor}.${patch}"
    }
}

// Sonarscanner Function
def sonarScanner(Map args) {
    echo "Sonarqube Test"
    def serviceTypeCode = "${args.service_code}"
    echo "Service Code Type: ${serviceTypeCode}"

    if (serviceTypeCode == "maven"){
        sh """ /opt/apache-maven-3.9.5/bin/mvn --settings /opt/settings.xml clean verify sonar:sonar \
            -Dsonar.projectName=${args.service_name} \
            -Dsonar.projectKey=${args.app_name} \
            -Dsonar.projectVersion=${args.version} \
            -Dsonar.qualitygate=${args.quality_gate_name} \
            -Dsonar.qualitygate.wait=${args.quality_gate}\
            -Dsonar.branch.name=${args.sonarBranch}
        """
    } else {
        echo "test"
        sh """ /opt/sonar-scanner/bin/sonar-scanner \
            -Dsonar.projectName=${args.service_name} \
            -Dsonar.projectKey=${args.app_name} \
            -Dsonar.projectVersion=${args.version} \
            -Dsonar.qualitygate=${args.quality_gate_name} \
            -Dsonar.qualitygate.wait=${args.quality_gate}\
            -Dsonar.sources=${args.sonarSrc} \
            -Dsonar.c.file.suffixes=- \
            -Dsonar.cpp.file.suffixes=- \
            -Dsonar.objc.file.suffixes=- \
            -Dsonar.branch.name=${args.sonarBranch}
        """
    }
}

// Docker Function
def dockerBuild(Map args) {
    sh "docker build  --build-arg http_proxy=${http_proxy} --build-arg https_proxy=${https_proxy} --no-cache -t ${args.image_name}:${args.image_version} . -f ../dockerfile/${args.service_type}/${args.service_name}/Dockerfile"
}

def dockerPushTag(Map args) {
    sh "docker tag ${args.image_name}:${args.image_version} ${args.image_name}:${args.dst_version}"
    sh "docker push ${args.image_name}:${args.dst_version}"
}

def dockerPush(Map args) {
    sh "docker push ${args.image_name}:${args.image_version}"
}

def dockerPull(Map args) {
    sh "docker pull ${args.image_name}:${args.image_version}"
}

def dockerRemove(Map args) {
    sh "docker rmi -f ${args.image_name}:${args.image_version}"
}

def dockerBackup(Map args) {
    sh "docker pull ${args.image_name}:${args.image_version}"
    sh "docker tag ${args.image_name}:${args.image_version} ${args.image_name}:revert"
    sh "docker push ${args.image_name}:revert"
    sh "docker rmi -f ${args.image_name}:${args.image_version}"
    sh "docker rmi -f ${args.image_name}:revert"
}

def dockerUpdateImage(Map args) {
    def version             = "${args.version}"
    def docker_url_source, docker_url_target
    if (version == "production"){
        docker_url_source   = "registry_url/staging/${args.app_name}/${args.service_type}/${args.service_name}"
        docker_url_target   = "registry_url/${args.app_name}/${args.service_type}/${args.service_name}"

    } else {
        docker_url_source   = "registry_url/test/${args.app_name}/${args.service_type}/${args.service_name}"
        docker_url_target   = "registry_url/staging/${args.app_name}/${args.service_type}/${args.service_name}"
    }

    dockerPull(image_name: docker_url_source, image_version: "${args.sourceTag}")
    sh "docker tag ${docker_url_source}:${args.sourceTag} ${docker_url_target}:${args.targetTag}"
    dockerPush(image_name: docker_url_target, image_version: "${args.targetTag}")
    dockerRemove(image_name: docker_url_target, image_version: "${args.targetTag}")
    dockerRemove(image_name: docker_url_source, image_version: "${args.sourceTag}")
}

// GIT Function
def getProjectId(Map args) {
    def apiUrl = "https://git_url/api/v4/projects/pti%2F${args.app_name}%2F${args.service_type}%2F${args.service_name}"
    def response = sh(script: """https_proxy=${http_proxy} curl -svf --header "PRIVATE-TOKEN: " "$apiUrl" """, returnStdout: true).trim()

    // Parse JSON
    def jsonSlurper = new groovy.json.JsonSlurper()
    def result = jsonSlurper.parseText(response)

    // Find merge request IID (Internal ID)
    def projectIdRepo = result.id

    // Check if projectIdRepo is not null or empty
    if (projectIdRepo == null || projectIdRepo.toString().trim().isEmpty()) {
        echo "Error: projectIdRepo is either null or empty!"
        return null
    }

    return projectIdRepo
}


def getLatestMergeRequestId(Map args) {
    def apiUrl = "https://git_url/api/v4/projects/${args.project_id}/merge_requests?source_branch=${args.source_repo}&state=opened"
    def response = sh(script: """https_proxy=${http_proxy} curl -svf --header "PRIVATE-TOKEN: " "$apiUrl" """, returnStdout: true).trim()
    
    // Parse JSON
    def jsonSlurper = new groovy.json.JsonSlurper()
    def result = jsonSlurper.parseText(response)
    
    // Find merge request IID (Internal ID)
    def mergeRequestId = result[0]?.iid

    return mergeRequestId
}

def isMergeRequestApproved(Map args) {
    def projectId = "${args.project_id}"
    def sourceRepo = "${args.source_branch}"
    def approvers = args.approvers
    def treshold  = args.treshold
    def mergeRequestId = getLatestMergeRequestId(project_id: projectId, source_repo: sourceRepo)
    
    echo "Debug: Merge Request ID - ${mergeRequestId}"
    
    if (mergeRequestId == null) {
        echo "Warning: Merge Request ID not found."
        return false
    }
    
    return checkApprovalFromSpecificUser(mergeRequestId: mergeRequestId, project_id: projectId, approvers: approvers, treshold: treshold)
}

def checkApprovalFromSpecificUser(Map args) {
    def approvalMergeResults = true
    def tresholdList    = args.treshold
    def approversList   = args.approvers
    
    def apiUrl = "https://git_url/api/v4/projects/${args.project_id}/merge_requests/${args.mergeRequestId}/notes"
    def response = sh(script: """https_proxy=${http_proxy} curl -svf --header "PRIVATE-TOKEN: " "$apiUrl" """, returnStdout: true).trim()

    def jsonSlurper = new groovy.json.JsonSlurper()
    def result = jsonSlurper.parseText(response)
    String testingResponse = result.toString()
    echo testingResponse
    
    def approvedUsers = []

    for (def note in result) {
        if (note.body == 'Approved') {
            echo "Testing 133"
            String messageUsername = note.author.username
            String messageBody     = note.body
            def targetSubarray = [messageUsername, messageBody]
            def containsTargetSubarray = approvedUsers.any { it == targetSubarray }
            if (containsTargetSubarray) {
                echo "The array contains the target subarray: $targetSubarray"
            } else {
                echo "The array does not contain the target subarray: $targetSubarray"
                approvedUsers << [messageUsername, messageBody]
            }
        } 
    }
    
    for (subArrayListApproval in approvedUsers) {
        echo "Sub-array Approved: ${subArrayListApproval}"
    }
    
    for (subArrayListApproval in args.treshold) {
        echo "Sub-array Threshold: ${subArrayListApproval}"
    }
    
    for (subArrayListApproval in args.approvers) {
        echo "Sub-array Approvers: ${subArrayListApproval}"
    }

    def thresholdDict = [:]
    def counterDict = [:]
    tresholdList.each { entry ->
        thresholdDict[entry[0]] = entry[1]
        counterDict[entry[0]] = 0
    }                     

    // Check each username for approval and threshold
    approversList.each { entry ->
        def department = entry[0]
        def username = entry[1]

        // Check if the username is approved
        def approvalStatus = approvedUsers.find { it[0] == username }?.get(1)
        echo approvalStatus
        echo "Testing 172"
        if ( approvalStatus != null && approvalStatus == 'Approved'){
            def numberCount = counterDict[department]
            counterDict.put(department, numberCount + 1)
            echo "Keluar berapa kali"
            echo "Number : ${numberCount}"
        }
    }

    tresholdList.each { entry ->
        def department = entry[0]
        def thresholdNumber = entry[1]

        def counterDepartmentCounter = counterDict[department]
        if (counterDepartmentCounter < thresholdNumber){
            approvalMergeResults = false
        }

        echo "$department has been approved by $counterDepartmentCounter from threshold $thresholdNumber"
    }

    return approvalMergeResults
}

def acceptMergeRequest(Map args) {
    def apiUrl = "https://git_url/api/v4/projects/${args.project_id}/merge_requests/${args.mergeRequestId}/merge"

    // Debug: Print API URL
    echo "Debug: API URL for merge - ${apiUrl}"

    def response = sh(script: """https_proxy=${http_proxy} curl -svf --request PUT --header "PRIVATE-TOKEN: " "$apiUrl" """, returnStdout: true).trim()

    echo "Merge Response: ${response}"
}


def isMergeRequestReviewed(Map args) {
    def projectId = "${args.project_id}"
    def sourceRepo = "${args.source_branch}"
    def reviews = args.reviews
    def treshold  = args.treshold
    echo "Project ID Test: ${projectId}"
    echo "Source Repo  Test: ${sourceRepo}"
    def mergeRequestId = getLatestMergeRequestId(project_id: projectId, source_repo: sourceRepo)
    echo "Line 1311"
    echo "Debug: Merge Request ID - ${mergeRequestId}"
    
    if (mergeRequestId == null) {
        echo "Warning: Merge Request ID not found."
        return false
    }
    
    return checkReviewedFromSpecificUser(mergeRequestId: mergeRequestId, project_id: projectId, approvers: reviews, treshold: treshold)
}

def checkReviewedFromSpecificUser(Map args) {
    def reviewMergeResults = true
    def tresholdList    = args.treshold
    def approversList   = args.approvers
    
    def apiUrl = "https://git_url/api/v4/projects/${args.project_id}/merge_requests/${args.mergeRequestId}/notes"
    def response = sh(script: """https_proxy=${http_proxy} curl -svf --header "PRIVATE-TOKEN: " "$apiUrl" """, returnStdout: true).trim()

    def jsonSlurper = new groovy.json.JsonSlurper()
    def result = jsonSlurper.parseText(response)
    String testingResponse = result.toString()
    echo testingResponse
    
    def approvedUsers = []

    for (def note in result) {
        if (note.body == 'Reviewed') {
            echo "Testing 133"
            String messageUsername = note.author.username
            String messageBody     = note.body
            def targetSubarray = [messageUsername, messageBody]
            def containsTargetSubarray = approvedUsers.any { it == targetSubarray }
            if (containsTargetSubarray) {
                echo "The array contains the target subarray: $targetSubarray"
            } else {
                echo "The array does not contain the target subarray: $targetSubarray"
                approvedUsers << [messageUsername, messageBody]
            }
        } 
    }
    
    for (subArrayListApproval in approvedUsers) {
        echo "Sub-array Approved: ${subArrayListApproval}"
    }
    
    for (subArrayListApproval in args.treshold) {
        echo "Sub-array Threshold: ${subArrayListApproval}"
    }
    
    for (subArrayListApproval in args.approvers) {
        echo "Sub-array Approvers: ${subArrayListApproval}"
    }

    def thresholdDict = [:]
    def counterDict = [:]
    tresholdList.each { entry ->
        thresholdDict[entry[0]] = entry[1]
        counterDict[entry[0]] = 0
    }                     

    // Check each username for approval and threshold
    approversList.each { entry ->
        def department = entry[0]
        def username = entry[1]

        // Check if the username is approved
        def approvalStatus = approvedUsers.find { it[0] == username }?.get(1)
        echo approvalStatus
        echo "Testing 172"
        if ( approvalStatus != null && approvalStatus == 'Reviewed'){
            def numberCount = counterDict[department]
            counterDict.put(department, numberCount + 1)
            echo "Keluar berapa kali"
            echo "Number : ${numberCount}"
        }
    }

    tresholdList.each { entry ->
        def department = entry[0]
        def thresholdNumber = entry[1]

        def counterDepartmentCounter = counterDict[department]
        if (counterDepartmentCounter < thresholdNumber){
            reviewMergeResults = false
        }

        echo "$department has been approved by $counterDepartmentCounter from threshold $thresholdNumber"
    }

    return reviewMergeResults

    return reviewMergeResults
}

def noteCloseMergeRequest(Map args) {
    def apiUrl = "https://git_url/api/v4/projects/${args.project_id}/merge_requests/${args.mergeRequestId}/notes"

    // Debug: Print API URL
    echo "Debug: API URL for note close MR - ${apiUrl}"

    def response = sh(script: """https_proxy=${http_proxy} curl -svf --request POST --header "PRIVATE-TOKEN: " --header "Content-Type: application/json" --data-raw '{ \"body\": \"Merge Request Closed due to Error\"}' "$apiUrl" """, returnStdout: true).trim()

    echo "Merge Response: ${response}"
}

def closeMergeRequest(Map args) {
    def apiUrl = "https://git_url/api/v4/projects/${args.project_id}/merge_requests/${args.mergeRequestId}?state_event=close"

    // Debug: Print API URL
    echo "Debug: API URL for merge - ${apiUrl}"

    def response = sh(script: """https_proxy=${http_proxy} curl -svf --request PUT --header "PRIVATE-TOKEN: " "$apiUrl" """, returnStdout: true).trim()

    echo "Merge Response: ${response}"
}


def getLastMergedStatus(Map args) {
    def apiUrl = "https://git_url/api/v4/projects/${args.project_id}/merge_requests?state=all&order_by=created_at&sort=desc&per_page=1"
    def response = sh(script: """https_proxy=${http_proxy} curl -svf --header "PRIVATE-TOKEN: " "$apiUrl" """, returnStdout: true).trim()
    echo response
    // Parse JSON
    def jsonSlurper = new groovy.json.JsonSlurper()
    def result = jsonSlurper.parseText(response)

    // Find merge request IID (Internal ID)
    def mergeStatus = result[0].state

    return mergeStatus
}

def getLastMergedId(Map args) {
    def apiUrl = "https://git_url/api/v4/projects/${args.project_id}/merge_requests?state=all&order_by=created_at&sort=desc&per_page=1"
    def response = sh(script: """https_proxy=${http_proxy} curl -svf --header "PRIVATE-TOKEN: " "$apiUrl" """, returnStdout: true).trim()
    echo response
    // Parse JSON
    def jsonSlurper = new groovy.json.JsonSlurper()
    def result = jsonSlurper.parseText(response)

    // Find merge request IID (Internal ID)
    def mergeId = result[0].iid

    return mergeId
}

def mergedComment(Map args) {
    def apiUrl = "https://git_url/api/v4/projects/${args.project_id}/merge_requests/${args.mergeRequestId}/notes"

    // Debug: Print API URL
    echo "Debug: API URL for note close MR - ${apiUrl}"

    def response = sh(script: """https_proxy=${http_proxy} curl -svf --request POST --header "PRIVATE-TOKEN: " --header "Content-Type: application/json" --data-raw '{ \"body\": \"Error due to Manual Merge\"}' "$apiUrl" """, returnStdout: true).trim()

    echo "Merge Response: ${response}"
}

//Backup Function
def backupImageTag(Map args){
    def versionImage = "${args.version}"

    echo "Get Image SHA from ArgoCD"
    final String url = "${args.server}/api/v1/applications/${args.application}"
    final String method = "GET"
    final String authorization_header = "Authorization: Bearer ${args.token}"
    final String response = sh(script: "curl --insecure --location --request ${method} '${url}' --header '${authorization_header}'", returnStdout: true).trim()
    echo response

    def slurper = new groovy.json.JsonSlurperClassic()
    def result = slurper.parseText(response)
    def sha_images = result.status.summary.images[0]
    String result_image = sha_images
    echo "Image used on ArgoCD: ${result_image}"
    def imageSplit = result_image.split(':')
    def imageTag = imageSplit[1].toString()
    echo "Image tag for backup image : ${imageTag}"

    return imageTag
}

// Gitops Function
def digestId(Map args) {
    String digestId = sh(script: "docker images --digests --format '{{.Digest}}' --filter=reference=${args.image_name} | head -1", returnStdout: true).trim()
    return digestId
}

def getImageId(Map args){
    def versionImage = "${args.version}"

    echo "Get Image SHA from ArgoCD"
    final String url = "${args.server}/api/v1/applications/${args.application}"
    final String method = "GET"
    final String authorization_header = "Authorization: Bearer ${args.token}"
    final String response = sh(script: "curl --insecure --location --request ${method} '${url}' --header '${authorization_header}'", returnStdout: true).trim()
    echo response

    def slurper = new groovy.json.JsonSlurperClassic()
    def result = slurper.parseText(response)
    def sha_images = result.status.summary.images[0]
    if ( versionImage == "develop" || versionImage == "test"){
        String[] str
        str = sha_images.split('@')
        String result_image = str[1]
        return result_image
    } else {
        String result_image = sha_images
        return result_image
    }
}


def getHealthStatus(Map args){
    echo "Get Health Check Status from ArgoCD"
    final String url = "${args.server}/api/v1/applications/${args.application}"
    final String method = "GET"
    final String authorization_header = "Authorization: Bearer ${args.token}"
    final String response = sh(script: "curl --insecure --location --request GET '${url}' --header '${authorization_header}'", returnStdout: true).trim()
    // echo response

    def slurper = new groovy.json.JsonSlurperClassic()
    def result = slurper.parseText(response)
    def health_status = result.status.health.status
    echo "health_status : ${health_status}"
    return health_status
}

def getSyncStatus(Map args){
    echo "Get Sync Status from ArgoCD"
    final String url = "${args.server}/api/v1/applications/${args.application}"
    final String authorization_header = "Authorization: Bearer ${args.token}"
    final String response = sh(script: "curl --insecure --location --request GET '${url}' --header '${authorization_header}'", returnStdout: true).trim()
    // echo response

    def slurper = new groovy.json.JsonSlurperClassic()
    def result = slurper.parseText(response)
    def sync_status = result.status.sync.status
    echo "sync_status : ${sync_status}"
    return sync_status
}

def generateReleaseNotes(lastTag) {
    def changes = sh(script: "git log ${lastTag}..HEAD --oneline", returnStdout: true).trim()
    return "Release notes for ${getCurrentTag()}\n\n${changes}"
}

// Approval Function
def listApprovalReviewer(Map args) {
    def apiUrl = "${args.url}/v1/${args.action}?env=${args.environment}&app=${args.application}"
    final String response = sh(script: """curl -svf "$apiUrl" """, returnStdout: true).trim()
    
    def jsonSlurper = new groovy.json.JsonSlurper()
    def result = jsonSlurper.parseText(response)
    String testingResponse = result.toString()
    echo testingResponse

    def approvalUser = result.pic
    def approvalList = []
    
    for (def user in approvalUser) {
        approvalList << [user.jobdesc, user.gitAccount]
    }
    
    return approvalList
}

def listThresholdApprovalReviewer(Map args) {
    def apiUrl = "${args.url}/v1/${args.action}?env=${args.environment}&app=${args.application}"
    final String response = sh(script: """curl -svf "$apiUrl" """, returnStdout: true).trim()
    
    def jsonSlurper = new groovy.json.JsonSlurper()
    def result = jsonSlurper.parseText(response)
    String testingResponse = result.toString()
    echo testingResponse
    
    def tresholdCount = result.threshold
    def tresholdList = []
    
    for (def minTreshold in tresholdCount) {
        String jobdesc = minTreshold.jobdesc
        String threshold = minTreshold.minThreshold
        echo jobdesc
        echo threshold
        tresholdList << [minTreshold.jobdesc, minTreshold.minThreshold]
    }
    
    return tresholdList
}

