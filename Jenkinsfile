#!groovy
/*
 * Licensed to the Apache Software Foundation (ASF) under one
 * or more contributor license agreements.  See the NOTICE file
 * distributed with this work for additional information
 * regarding copyright ownership.  The ASF licenses this file
 * to you under the Apache License, Version 2.0 (the
 * "License"); you may not use this file except in compliance
 * with the License.  You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */

def AGENT_LABEL = env.AGENT_LABEL ?: 'ubuntu'
def JDK_NAME = env.JDK_NAME ?: 'jdk_11_latest'
def MAVEN_NAME = 'maven_3_latest'

pipeline {

    agent {
        label AGENT_LABEL
    }

    triggers {
        upstream(upstreamProjects: '../ApacheJames/master', threshold: hudson.model.Result.SUCCESS)
    }

    environment {
        CI = true
    }

    tools {
        jdk JDK_NAME
        maven MAVEN_NAME
    }

    options {
        // Configure an overall timeout for the build of one hour.
        timeout(time: 30, unit: 'MINUTES')
        // When we have test-fails e.g. we don't need to run the remaining steps
        skipStagesAfterUnstable()
        buildDiscarder(
                logRotator(artifactNumToKeepStr: '5', numToKeepStr: '10')
        )
        disableConcurrentBuilds()
    }

    stages {
        stage('Build') {
            steps {
                sh "./gradlew clean build"
                stash includes: 'doc-sites/build/site/**/*', name: 'apache-james-site'
            }
        }

        stage('Run tests') {
            steps {
                echo "No tests to see. Move along."
            }
        }

        stage('Publish') {
            agent {
                node {
                    label 'git-websites'
                }
            }
            when {
                anyOf { branch 'live'; branch 'staging' }
            }

            steps {
                echo "Deploy staging James website."

                unstash 'apache-james-site'

                sh 'mvn -f jenkins.pom -X -P deploy-site scm-publish:publish-scm'
            }
        }
    }

    // Do any post build stuff ... such as sending emails depending on the overall build result.
    post {
        // If this build failed, send an email to the list.
        failure {
            echo "Failed "
            script {
                if (env.BRANCH_NAME == "live" || env.BRANCH_NAME == "staging") {
                    emailext(
                            subject: "[BUILD-UNSTABLE]: Job '${env.JOB_NAME} [${env.BRANCH_NAME}] [${env.BUILD_NUMBER}]'",
                            body: """
        BUILD-UNSTABLE: Job '${env.JOB_NAME} [${env.BRANCH_NAME}] [${env.BUILD_NUMBER}]':

        Check console output at "<a href="${env.BUILD_URL}">${env.JOB_NAME} [${env.BRANCH_NAME}] [${env.BUILD_NUMBER}]</a>"
        """,
                            recipientProviders: [[$class: 'DevelopersRecipientProvider']]
                    )
                } else {
                    emailext(
                            subject: "[BUILD-UNSTABLE]: Job '${env.JOB_NAME} [${env.BRANCH_NAME}] [${env.BUILD_NUMBER}]'",
                            body: """
        BUILD-UNSTABLE: Job '${env.JOB_NAME} [${env.BRANCH_NAME}] [${env.BUILD_NUMBER}]':

        Check console output at "<a href="${env.BUILD_URL}">${env.JOB_NAME} [${env.BRANCH_NAME}] [${env.BUILD_NUMBER}]</a>"
        """,
                            recipientProviders: [[$class: 'RequesterRecipientProvider']]
                    )
                }
            }

        }

        // If this build didn't fail, but there were failing tests, send an email to the list.
        unstable {
            echo "Unstable "
        }

        // Send an email, if the last build was not successful and this one is.
        success {
            echo "Success "
            script {
                if (env.BRANCH_NAME == "live" || env.BRANCH_NAME == "staging") {

                    if ((currentBuild.previousBuild != null) && (currentBuild.previousBuild.result != 'SUCCESS')) {
                        emailext(
                                subject: "[BUILD-STABLE]: Job '${env.JOB_NAME} [${env.BRANCH_NAME}] [${env.BUILD_NUMBER}]'",
                                body: """
    BUILD-STABLE: Job '${env.JOB_NAME} [${env.BRANCH_NAME}] [${env.BUILD_NUMBER}]':

    Is back to normal.
    """,
                                recipientProviders: [[$class: 'DevelopersRecipientProvider']]
                        )
                    }
                } else {
                    if ((currentBuild.previousBuild != null) && (currentBuild.previousBuild.result != 'SUCCESS')) {
                        emailext(
                                subject: "[BUILD-STABLE]: Job '${env.JOB_NAME} [${env.BRANCH_NAME}] [${env.BUILD_NUMBER}]'",
                                body: """
    BUILD-STABLE: Job '${env.JOB_NAME} [${env.BRANCH_NAME}] [${env.BUILD_NUMBER}]':

    Is back to normal.
    """,
                                recipientProviders: [[$class: 'RequesterRecipientProvider']]
                        )
                    }
                }
            }
        }

        always {
            echo "Build done"
        }
    }
}
