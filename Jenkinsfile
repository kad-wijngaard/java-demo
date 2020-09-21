#!/usr/bin/env groovy

final String branch = env.BRANCH_NAME.toLowerCase()
def allowedBranchesPattern = /^tms-.*$/

def buildEnv = [
        dockerLabel: 'Docker 18.06',
        jdkLabel   : 'OpenJDK 8',
        maven      : 'Maven 3.6',
]

String baseRegistryName, orgName, repoName, fullImageName, version, versionNumber, releaseOption, releasedTag, newReleaseVersion, commitId
switch (branch) {
    case ['master']:
        baseRegistryName = env.DOCKER_PROD_URL
        break
    default:
        baseRegistryName = env.DOCKER_DEV_URL
        break
}
String fullRegistryUrl = "https://${baseRegistryName}"

pipeline {
    agent any

    options {
        disableConcurrentBuilds()
        timeout(time: 1, unit: 'HOURS')
    }

    parameters {
        choice(
                name: 'releaseOption',
                choices: "none\npatch\nminor\nmajor",
                description: "Specify which kind of release should be released."
        )
        string(
                name: 'releasedTag',
                defaultValue: "none",
                description: "Specify which tag to merge."
        )
    }

    tools {
        jdk buildEnv['jdkLabel']
        maven buildEnv['maven']
    }


    stages {

        stage("Git checkout tag") {
            steps {
                script {
                    echo "Git checkout of tag..."
                    checkout([$class                           : 'GitSCM',
                              branches                         : scm.branches,
                              doGenerateSubmoduleConfigurations: false,
                              extensions                       : scm.extensions + [[$class: 'LocalBranch'], [$class: 'WipeWorkspace']],
                              userRemoteConfigs                : [[credentialsId: 'jenkins-up', url: "${env.GIT_URL}"]]])
                }
            }
        }

        stage('Init') {
            steps {
                script {
                    if (!'master'.contains(branch) && !'development'.contains(branch) && !(branch =~ allowedBranchesPattern)) {
                        currentBuild.result = 'ABORTED'
                        error("Error: Tried to build branch: ${branch}. But only master, development, feature and hotfix branches allowed")
                    }

                    echo "Initializing..."
                    echo "Determining organisation and repository name"
                    def remoteParts = scm.getUserRemoteConfigs()[0].getUrl().tokenize('/')
                    orgName = "pmk"
                    repoName = remoteParts[remoteParts.size() - 1].split("\\.")[0] as String

                    echo "Determining image name"
                    final String baseImageName = "${orgName}/${repoName}"
                    fullImageName = "${baseRegistryName}/${baseImageName}"

                    echo "Setting Docker credentials"
                    credentialsId = env.DOCKER_BUILD_HOST.tokenize('.')[0]

                    echo "Getting version from pom"
                    pom = readMavenPom file: 'pom.xml'
                    version = pom.version

                    echo "set releasOption & releasedTag value"
                    releaseOption = params.releaseOption
                    releasedTag = params.releasedTag

                    commitId = sh(script: 'git log -1 -g --date=format:\'%Y%m%d%H%M\' --pretty=format:%cd-%h HEAD', returnStdout: true).trim()

                    echo """
               Logging environmental variables:
               Application version: ${version}
               Branch: ${branch}
               Registry URL: ${baseRegistryName}
               Organization: ${orgName}
               Repository: ${repoName}
               Base Image Name: ${baseImageName}
               Full Image Name: ${fullImageName}
               Docker Label: ${buildEnv['dockerLabel']}
               JDK Label: ${buildEnv['jdkLabel']}
             """
                }
            }
        }

        stage('build') {
            steps {
                sh "mvn clean install"
                echo 'Publish codecoverage report'
                publishHTML(target: [
                        allowMissing         : false,
                        alwaysLinkToLastBuild: false,
                        keepAll              : false,
                        reportDir            : 'target/..',
                        reportFiles          : 'target/site/jacoco/index.html',
                        reportName           : 'Code Coverages'
                ])
                cucumber fileIncludePattern: 'target/cucumber.json', sortingMethod: 'ALPHABETICAL'
            }
        }
        stage('dependency report') {
            steps {
                sh "mvn versions:dependency-updates-report -DprocessDependencyManagement=false"
                echo 'Publish dependency report'
                publishHTML(target: [
                        allowMissing         : false,
                        alwaysLinkToLastBuild: false,
                        keepAll              : false,
                        reportDir            : 'target/..',
                        reportFiles          : 'target/site/dependency-updates-report.html',
                        reportTitles         : 'WEB Dependencys',
                        reportName           : 'State of Dependencies'
                ])
            }
        }

        stage('Docker build') {
            steps {
                script {

                    docker.withTool(buildEnv['dockerLabel']) {
                        docker.withServer("tcp://${env.DOCKER_BUILD_HOST}:2376", credentialsId) {
                            echo "Building docker image..."
                            echo "Determining git commit info"

                            if (branch == 'master') {
                                dockerTag = "${commitId}_${branch}_V${version}"
                            } else {
                                dockerTag = "${commitId}_${branch}"
                            }

                            echo "Determining docker tag"
                            fullDockertag = "${fullImageName}:${dockerTag}"
                            echo "Using Docker Tag: ${fullDockertag}"

                            echo "Building image..."
                            buildImage = docker.build(fullDockertag, "--build-arg REGISTRY=${env.DOCKER_REGISTRY_URL} --label gitCommit=${commitId} .")

                            echo "Push to artifactory"
                            docker.withRegistry(fullRegistryUrl, 'jenkins-up') {
                                echo "Pushing with git commit tag"
                                buildImage.push()
                            }
                        }
                    }
                }
            }
        }


//        // Sonar Step
//        stage("Temporary shell command for certificates") {
//            steps {
//                sh "cp /etc/ssl/certs/java/cacerts ${JAVA_HOME}/jre/lib/security/cacerts"
//            }
//        }
//
//        stage("SonarQube analysis") {
//            steps {
//                withSonarQubeEnv('SonarQube') {
//                    sh "mvn sonar:sonar -Dsonar.branch.name=${branch}"
//                }
//            }
//        }

    }
    post {
        always {
            // Display Unit test reports
            junit '**/target/*-reports/TEST-*.xml'
        }
    }
}


