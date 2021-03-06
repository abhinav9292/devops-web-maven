#!/usr/bin/env groovy

/**
 * @ Maintainer sudheer veeravalli<veersudhir83@gmail.com>
 * Plugins Needed in Jenkins: Ant, Maven, Ansible, Artifactory, SonarQube,
 * PMD, JUnit, Jacoco, HTML Publisher,
 */

/* Only keep the 10 most recent builds. */
def projectProperties = [
        buildDiscarder(logRotator(artifactDaysToKeepStr: '20', artifactNumToKeepStr: '20', daysToKeepStr: '20', numToKeepStr: '20')),
        [$class: 'GithubProjectProperty', projectUrlStr: 'https://github.com/veersudhir83/devops-web-maven.git/']
        //,pipelineTriggers([pollSCM('H/10 * * * *')])
]

properties(projectProperties)

try {
    node {
        def mvnHome
        def mvnAnalysisTargets = '-P metrics pmd:pmd test javadoc:javadoc '
        def antHome
        def artifactoryPublishInfo
        def artifactoryDownloadInfo
        def artifactoryServer
        def isArchivalEnabled = true //params.IS_ARCHIVAL_ENABLED // Enable if you want to archive files and configs to artifactory
        def isSonarAnalysisEnabled = true //params.IS_ANALYSIS_ENABLED // Enable if you want to analyze code with sonarqube
        def isDeploymentEnabled = true //params.IS_DEPLOYMENT_ENABLED // Enable if you want to deploy code on app server
        def isSeleniumTestingEnabled = true //params.IS_SELENIUM_TESTING_ENABLED // Enable if you want to generate reports
        def isReportsEnabled = true //params.IS_REPORTS_ENABLED // Enable if you want to generate reports

        def appName = 'devops-web-maven'// application name currently in progress
        def appEnv  // application environment currently in progress
        def artifactName = appName // name of the war/jar/ear in artifactory
        def artifactExtension = "jar" // extension of the war/jar/ear - for both target directory and artifactory
        def artifactoryRepoName = 'DevOps' // repo name in artifactory
        def artifactoryAppName = appName // application name as per artifactory

        def buildNumber = env.BUILD_NUMBER
        def workspaceRoot = env.WORKSPACE
        def artifactoryTempFolder = 'downloadsFromArtifactory' // name of the local temp folder where file(s) from artifactory get downloaded
        def sonarHome
        def SONAR_HOST_URL = 'http://localhost:9000'

        // Logic for Slack Notification Service
        def slackBaseUrl = 'https://defaultgrouptalk.slack.com/services/hooks/jenkins-ci/'
        def slackChannel = '#general'
        def slackTeamDomain = 'defaultgrouptalk'
        def slackMessagePrefix = "Job ${env.JOB_NAME}:${env.BUILD_NUMBER}"
        def slackTokenCredentialId = 'ecd292a7-bf0e-45c9-b599-aeb317ce2170' // replace with right one from jenkins credentials details

        // color can be good, warning, danger or anything
        slackSend baseUrl: "${slackBaseUrl}", channel: "${slackChannel}", color: "good", message: "${slackMessagePrefix} -> Build Started", teamDomain: "${slackTeamDomain}", tokenCredentialId: "${slackTokenCredentialId}"


        if (isArchivalEnabled) {
            // Artifactory server id configured in the jenkins along with credentials
            artifactoryServer = Artifactory.server 'Artifactory'
        }

        // to download appConfig.json files from artifactory
        def downloadAppConfigUnix = """{
            "files": [
                {
                    "pattern": "generic-local/Applications/${artifactoryRepoName}/${artifactoryAppName}/${appEnv}/appConfig.json",
                    "target": "${workspaceRoot}/${artifactoryTempFolder}/"
                }
            ]
        }"""

        def downloadAppConfigWindows = """{
            "files": [
                {
                    "pattern": "generic-local/Applications/${artifactoryRepoName}/${artifactoryAppName}/${appEnv}/appConfig.json",
                    "target": "${workspaceRoot}/${artifactoryTempFolder}/"
                }
            ]
        }"""

        def uploadAppConfigUnix = """{
            "files": [
                {
                    "pattern": "${workspaceRoot}/${artifactoryTempFolder}/${artifactoryAppName}/${appEnv}/appConfig.json",
                    "target": "generic-local/Applications/${artifactoryRepoName}/${artifactoryAppName}/${appEnv}/"
                }
            ]
        }"""

        def uploadAppConfigWindows = """{
            "files": [
                {
                    "pattern": "${workspaceRoot}\\${artifactoryTempFolder}\\${artifactoryAppName}\\${appEnv}\\appConfig.json",
                    "target": "generic-local/Applications/${artifactoryRepoName}/${artifactoryAppName}/${appEnv}/"
                }
            ]
        }"""

        def uploadMavenArtifactUnix = """{
            "files": [
                {
                    "pattern": "${workspaceRoot}/${appName}/target/${artifactName}.${artifactExtension}",
                    "target": "generic-local/Applications/${artifactoryRepoName}/${artifactoryAppName}/app/${buildNumber}/"
                }
            ]
        }"""

        def uploadMavenArtifactWindows = """{
            "files": [
                {
                    "pattern": "${workspaceRoot}\\${appName}\\target\\${artifactName}.${artifactExtension}",
                    "target": "generic-local/Applications/${artifactoryRepoName}/${artifactoryAppName}/app/${buildNumber}/"
                }
            ]
        }"""

        stage('Tool Setup'){

            // ** NOTE: These tools must be configured in the jenkins global configuration.
            try {
                if (isUnix()) {
                    sh "echo 'Running in Unix mode'"
                } else {
                    bat(/echo 'Running in windows mode' /)
                }
                mvnHome = tool name: 'mvn', type: 'maven'
                antHome = tool name: 'ant', type: 'ant'
                ansible = tool name: 'ansible', type: 'org.jenkinsci.plugins.ansible.AnsibleInstallation'

                if (isSonarAnalysisEnabled) {
                    sonarHome = tool name: 'sonar-scanner', type: 'hudson.plugins.sonar.SonarRunnerInstallation'
                }
                slackSend color: "good", message: "${slackMessagePrefix} -> Tool Setup Complete"

            } catch (exc) {
                slackSend color: "danger", message: "${slackMessagePrefix} -> Tool Setup Failed"
                error "Failure in Tool Setup stage: ${exc}"
            }
        }

        stage('Checkout') {
            try {
                //cleanWs() // cleanup workspace before build starts

                // Checkout codes from repository
                dir('devops-web-maven') {
                    git url: 'https://github.com/veersudhir83/devops-web-maven.git',
                            branch: 'master'
                }

                if (isSeleniumTestingEnabled) {
                    dir('devops-web-test-suite') {
                        git url: 'https://github.com/veersudhir83/devops-web-test-suite.git',
                                branch: 'master'
                    }
                }

                dir('downloadsFromArtifactory') {
                    // created folder for artifactory
                }

                slackSend color: "good", message: "${slackMessagePrefix} -> Checkout Complete"
            } catch (exc) {
                slackSend color: "danger", message: "${slackMessagePrefix} -> Checkout Failed"
                error "Failure in Checkout stage: ${exc}"
            }
        }

        stage('Build') {
            try {
                if (isUnix()) {
                    dir('devops-web-maven/') {
                        sh "'${mvnHome}/bin/mvn' clean package"
                        sh "cp ./target/${appName}*.${artifactExtension} ./target/${appName}.${artifactExtension}"
                    }
                } else {
                    dir('devops-web-maven\\') {
                        bat(/"${mvnHome}\bin\mvn" --batch-mode clean package/)
                        bat(/copy .\\target\\\u0024{appName}*.\u0024{artifactExtension} .\\target\\\u0024{appName}.\u0024{artifactExtension}/)
                    }
                }
                slackSend color: "good", message: "${slackMessagePrefix} -> Build Complete"
            } catch (exc) {
                slackSend color: "danger", message: "${slackMessagePrefix} -> Build Failed"
                error "Failure in Build stage: ${exc}"
            }
        }

        stage('Analysis') {
            try {
                if (isSonarAnalysisEnabled) {
                    mvnAnalysisTargets = "${mvnAnalysisTargets} sonar:sonar"
                }
                if (isUnix()) {
                    dir('devops-web-maven/') {
                        sh "'${mvnHome}/bin/mvn' ${mvnAnalysisTargets}"
                    }
                } else {
                    dir('devops-web-maven\\') {
                        bat(/"${mvnHome}\bin\mvn" --batch-mode ${mvnAnalysisTargets} /)
                    }
                }
                slackSend color: "good", message: "${slackMessagePrefix} -> Analysis Complete"
            } catch (exc) {
                slackSend color: "danger", message: "${slackMessagePrefix} -> Analysis Failed"
                error "Failure in Analysis stage: ${exc}"
            }
        }

        stage('Publish') {
            try {
                if (isArchivalEnabled) {
                    echo 'Publish Artifacts & appConfig.json in progress'
                    if (isUnix()) {
                        dir('devops-web-maven/') {
                            if (fileExists('target/devops-web-maven.jar')) {
                                // upload artifactory and also publish build info
                                artifactoryPublishInfo = artifactoryServer.upload(uploadMavenArtifactUnix)
                                artifactoryPublishInfo.retention maxBuilds: 5
                                // and publish build info to artifactory
                                artifactoryServer.publishBuildInfo(artifactoryPublishInfo)
                            } else {
                                error 'Publish: Failed during file upload/publish to artifactory'
                            }
                        }
                        // TODO: Work on this
                        /*causing Error: Error occurred for request CONNECT localhost:8081 HTTP/1.1:
                        sun.security.validator.ValidatorException: PKIX path building failed:
                        sun.security.provider.certpath.SunCertPathBuilderException:
                        unable to find valid certification path to requested target.*/

                        /*
                        artifactoryServer.download(downloadAppConfigUnix)
                        dir('downloadsFromArtifactory/') {
                            sh '''
                                curl -uadmin:APTvW3dVn6kUTbS -O "http://localhost:8081/artifactory/generic-local/Applications/DevOps/devops-web-maven/DEV/appConfig.json"
                                FILE=appConfig.json
                                TEMP=temp.json
                                if [ -f $FILE ]
                                then
                                echo "File $FILE exists."
                                mv $FILE $TEMP
                                command_publish="jq '.component[0].Build_Number = ${BUILD_NUMBER}' $TEMP > $FILE"
                                eval $command_publish
                                fi
                            '''
                        }
                        */
                    } else {
                        dir('devops-web-maven\\') {
                            if (fileExists('target\\devops-web-maven.jar')) {
                                // upload artifactory and also publish build info
                                artifactoryPublishInfo = artifactoryServer.upload(uploadMavenArtifactWindows)
                                artifactoryPublishInfo.retention maxBuilds: 5
                                // and publish build info to artifactory
                                artifactoryServer.publishBuildInfo(artifactoryPublishInfo)
                            } else {
                                error 'Publish: Failed during file upload/publish to artifactory'
                            }
                        }
                    }
                }
                slackSend color: "good", message: "${slackMessagePrefix} -> Archival Complete"
            } catch (exc) {
                slackSend color: "danger", message: "${slackMessagePrefix} -> Archival Failed"
                error "Failure in Publish stage: ${exc}"
            }
        }

        stage('Deployment') {
            if (isDeploymentEnabled) {
                echo 'Deploy application using ansible'
                try {
                    if (isUnix()) {
                        ansiblePlaybook installation: 'ansible1.5', playbook: 'devops-web-maven/configuration_scripts/app-service-deploy.yml'

                        // Deployment to docker containers devopsmaven-container*
                        // Commented out on purpose - Use with customization when needed
                        /*sh '''
                            docker ps -a | awk '{print $NF}' | grep -w devopsmaven* > temp.txt
                            sort temp.txt -o container_names_files.txt
                            while IFS='' read -r line || [[ -n "$line" ]]; do
                                echo "#############################"
                                STATUS=`docker inspect --format='{{json .State.Running}}' $line`
                                echo "Container Name is : $line and Status is $STATUS"

                                # copy war file to container
                                docker cp ./target/devops-web-maven.jar $line:/home/
                                echo "jar file is copied !!"
                            done < "container_names_files.txt"
                        '''*/
                    } else {
                        dir('devops-web-maven\\') {
                            // Do Something else
                        }
                    }
                    slackSend color: "good", message: "${slackMessagePrefix} -> Deployment Complete"
                } catch (exc) {
                    slackSend color: "danger", message: "${slackMessagePrefix} -> Deployment Failed"
                    error "Failure in Deployment stage: ${exc}"
                }
            }
        }

        stage('Build and Run Test Suite') {
            if (isSeleniumTestingEnabled) {
                try {
                    if (isUnix()) {
                        dir('devops-web-test-suite/build/') {
                            sh "'${antHome}/bin/ant'"
                            sh '''
                                chown -R jenkins:jenkins *
                                chmod 777 -R *
						    '''
                            wrap([$class: 'Xvfb', additionalOptions: '', assignedLabels: '', displayName: 99, displayNameOffset: 0, installationName: 'Default', screen: '1024x768x8', timeout: 20]) {
                                sh '''
								    # Xvfb :99 -screen 0 1024x768x8 > /dev/null
	                                java -jar test.jar LINUX CHROME
	                            '''
                            }
                        }
                    } else {
                        dir('devops-web-test-suite\\build\\') {
                            bat(/"java -jar test.jar WINDOWS FIREFOX"/)
                        }
                    }
                    slackSend color: "good", message: "${slackMessagePrefix} -> Test Suite Run Complete"
                } catch (exc) {
                    slackSend color: "danger", message: "${slackMessagePrefix} -> Test Suite Run Failed"
                    error "Failure in Build and Run Test Suite stage: ${exc}"
                }
            }
        }

        stage('Generate Reports') {
            if (isReportsEnabled) {
                try {
                    if (isUnix()) {
                        //jacoco()
                        junit '**/devops-web-maven/target/surefire-reports/*.xml'
                        //publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'devops-web-maven/target/site/apidocs', reportFiles: 'index.html', reportName: 'HTML Report', reportTitles: ''])
                    } else {
                        //jacoco()
                        junit '**\\devops-web-maven\\target\\surefire-reports\\*.xml'
                        //publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'devops-web-maven\\target\\site\\apidocs', reportFiles: 'index.html', reportName: 'HTML Report', reportTitles: ''])
                    }
                    slackSend color: "good", message: "${slackMessagePrefix} -> Generate Reports Complete"
                } catch (exc) {
                    slackSend color: "warning", message: "${slackMessagePrefix} -> Generate Reports Failed"
                    error "Failure in Generate Reports stage: ${exc}"
                }
            }
        }

    }
} catch (exc) {
    error "Caught: ${exc}"
}
