#!groovy

// Copyright 2017 Intel Corporation
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//     http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.
// ------------------------------------------------------------------------------

// Discard old builds after 31 days
properties([[$class: 'BuildDiscarderProperty', strategy:
        [$class: 'LogRotator', artifactDaysToKeepStr: '',
        artifactNumToKeepStr: '', daysToKeepStr: '31', numToKeepStr: '']]]);

node ('master') {
    // Create a unique workspace so Jenkins doesn't reuse an existing one
    ws("workspace/${env.BUILD_TAG}") {
        stage("Clone Repo") {
            checkout scm
            sh 'git fetch --tag'
        }

        if (!(env.BRANCH_NAME == 'master' && env.JOB_BASE_NAME == 'master')) {
            stage("Check Whitelist") {
                readTrusted 'bin/whitelist'
                sh './bin/whitelist "$CHANGE_AUTHOR" /etc/jenkins-authorized-builders'
            }
        }

        stage("Check for Signed-Off Commits") {
            sh '''#!/bin/bash -l
                if [ -v CHANGE_URL ] ;
                then
                    temp_url="$(echo $CHANGE_URL |sed s#github.com/#api.github.com/repos/#)/commits"
                    pull_url="$(echo $temp_url |sed s#pull#pulls#)"

                    IFS=$'\n'
                    for m in $(curl -s "$pull_url" | grep "message") ; do
                        if echo "$m" | grep -qi signed-off-by:
                        then
                          continue
                        else
                          echo "FAIL: Missing Signed-Off Field"
                          echo "$m"
                          exit 1
                        fi
                    done
                    unset IFS;
                fi
            '''
        }

        // Set the ISOLATION_ID environment variable for the whole pipeline
        env.ISOLATION_ID = sh(returnStdout: true, script: 'printf $BUILD_TAG | sha256sum | cut -c1-64').trim()
        // Set the project name for the whole pipeline
        env.COMPOSE_PROJECT_NAME = sh(returnStdout: true, script: 'printf $BUILD_TAG | sha256sum | cut -c1-64').trim()

        stage("Build Test Dependencies") {
            sh 'docker-compose -f examples/docker-compose.yaml up --abort-on-container-exit --force-recreate --renew-anon-volumes --exit-code-from intkey-tp intkey-tp'
            sh 'docker-compose -f examples/docker-compose.yaml up --abort-on-container-exit --force-recreate --renew-anon-volumes --exit-code-from xo-tp xo-tp'
        }

        // Run the tests
        stage("Run Tests") {
            sh 'docker-compose -f examples/intkey/tests/tp_tests_compose.yaml up --abort-on-container-exit --force-recreate --renew-anon-volumes --exit-code-from intkey-tests'
            sh 'docker-compose -f examples/intkey/tests/smoke_tests_compose.yaml up --abort-on-container-exit --force-recreate --renew-anon-volumes --exit-code-from intkey-tests'
            sh 'docker-compose -f examples/xo/tests/tp_tests_compose.yaml up --abort-on-container-exit --force-recreate --renew-anon-volumes --exit-code-from xo-tests'
            sh 'docker-compose -f examples/xo/tests/smoke_tests_compose.yaml up --abort-on-container-exit --force-recreate --renew-anon-volumes --exit-code-from xo-tests'
        }

        // Build the docs
        stage("Build docs") {
            sh 'docker build . -f docs/Dockerfile -t sawtooth-sdk-javascript-docs'
            sh 'docker run --rm -v $(pwd):/sawtooth-sdk-javascript sawtooth-sdk-javascript-docs'
        }

        stage("Create git archive") {
            sh '''
                REPO=$(git remote show -n origin | grep Fetch | awk -F'[/.]' '{print $6}')
                VERSION=`git describe --dirty`
                git archive HEAD --format=zip -9 --output=$REPO-$VERSION.zip
                git archive HEAD --format=tgz -9 --output=$REPO-$VERSION.tgz
            '''
        }

        stage("Archive Build artifacts") {
            archiveArtifacts artifacts: '*.tgz, *.zip'
            archiveArtifacts artifacts: 'docs/build/html/**'
        }
    }
}
