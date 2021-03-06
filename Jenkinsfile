/*
 * Pipeline for running integration tests in Jenkins job with 
 * every pull request or change in master
 *
 * Requires: https://github.com/RedHatInsights/insights-pipeline-lib
 */

@Library("github.com/RedHatInsights/insights-pipeline-lib") _

// Code coverage failure threshold
codecovThreshold = 89

node {
    // Cancel any prior builds that are running for this job
    cancelPriorBuilds()
    lock("vmaas-qe-deploy") {
        runStages()
    }
}


def runStages() {
    // withNode is a helper to spin up a jnlp slave using the Kubernetes plugin, and run the body code on that slave
    openShift.withNode(image: "docker-registry.default.svc:5000/jenkins/jenkins-slave-vmaas:latest") {
        // check out source again to get it in this node"s workspace
        scmVars = checkout scm

        // checkout vmaas_tests git repository
        checkOutRepo(targetDir: "vulnerability", repoUrl: "https://github.com/RedHatInsights/vmaas_tests", credentialsId: "github")

        stage("Pip install") {
            // withStatusContext runs the body code and notifies GitHub on whether it passed or failed
            withStatusContext.pipInstall {
                sh "pip install -U iqe-integration-tests pytest-html"
                sh "iqe plugin install vulnerability"
                try {
                    sh "pip check"
                } catch (err) {
                    echo err.toString()
                    // workaround for old pinned version in iqe-tests
                    sh "pip install -U simple-rest-client"
                }
            }
        }

        stage('Inject local settings') {
            withCredentials([file(credentialsId: "vmaas-settings-local-yaml", variable: "settings")]) {
                sh "cp ${settings} ${IQE_VENV}/lib/python3.6/site-packages/iqe_vulnerability/conf"
            }
        }

        stage("Deploy VMaaS") {
            // Deploy VMaaS to vmaas-qe project
            withStatusContext.custom("deploy") {
                stage("Login as deployer account") {
                    withCredentials([string(credentialsId: "openshift_token", variable: "TOKEN")]) {
                        sh "oc login https://${pipelineVars.devCluster} --token=${TOKEN}"
                    }
                    sh "oc project vmaas-qe"
                }

                checkOutRepo(targetDir: pipelineVars.e2eDeployDir, repoUrl: pipelineVars.e2eDeployRepoSsh, credentialsId: pipelineVars.gitSshCreds)
                dir(pipelineVars.e2eDeployDir) {
                    sh "${IQE_VENV}/bin/pip install -r requirements.txt"
                    // wipe old deployment
                    sh "${IQE_VENV}/bin/ocdeployer wipe -f vmaas-qe -l app=vmaas"
                    // make sure that DB volume is deleted
                    sh "oc delete pvc vmaas-db-data || true"
                    // git reference
                    String GIT_REF = "${env.BRANCH_NAME}"
                    if (env.BRANCH_NAME.startsWith("PR")) {
                        String GIT_PR = "${env.BRANCH_NAME}" - "PR-"
                        GIT_REF = "+refs/pull/${GIT_PR}/head"
                    }
                    // set needed env.yml
                    // build vmaas-webapp from Dockerfile-webapp-qe - it won't start app.py
                    sh """
                        # Create an env.yaml to have the builder pull from a different branch
                        echo "vmaas/vmaas-apidoc:" > builder-env.yml
                        echo "  parameters:" >> builder-env.yml
                        echo "    SOURCE_REPOSITORY_REF: ${GIT_REF}" >> builder-env.yml
                        echo "    SOURCE_REPOSITORY_URL: ${scmVars.GIT_URL}" >> builder-env.yml
                        echo "vmaas/vmaas-reposcan:" >> builder-env.yml
                        echo "  parameters:" >> builder-env.yml
                        echo "    SOURCE_REPOSITORY_REF: ${GIT_REF}" >> builder-env.yml
                        echo "    SOURCE_REPOSITORY_URL: ${scmVars.GIT_URL}" >> builder-env.yml
                        echo "vmaas/vmaas-webapp:" >> builder-env.yml
                        echo "  parameters:" >> builder-env.yml
                        echo "    SOURCE_REPOSITORY_REF: ${GIT_REF}" >> builder-env.yml
                        echo "    SOURCE_REPOSITORY_URL: ${scmVars.GIT_URL}" >> builder-env.yml
                        echo "    DOCKERFILE_PATH: webapp/Dockerfile-qe" >> builder-env.yml
                        echo "vmaas/vmaas-websocket:" >> builder-env.yml
                        echo "  parameters:" >> builder-env.yml
                        echo "    SOURCE_REPOSITORY_REF: ${GIT_REF}" >> builder-env.yml
                        echo "    SOURCE_REPOSITORY_URL: ${scmVars.GIT_URL}" >> builder-env.yml
                        echo "vmaas/vmaas-db:" >> builder-env.yml
                        echo "  parameters:" >> builder-env.yml
                        echo "    SOURCE_REPOSITORY_REF: ${GIT_REF}" >> builder-env.yml
                        echo "    SOURCE_REPOSITORY_URL: ${scmVars.GIT_URL}" >> builder-env.yml

                        # Deploy these customized builders into 'vmaas-qe' project
                        ${IQE_VENV}/bin/ocdeployer deploy -f --sets vmaas --template-dir buildfactory \
                            -e builder-env.yml vmaas-qe --secrets-local-dir secrets/sanitized
                    """
                    // deploy vmaas service set
                    sh "${IQE_VENV}/bin/ocdeployer deploy -f --sets vmaas \
                        --template-dir ../vulnerability/openshift/templates \
                        -e ../vulnerability/openshift/env/env.yml \
                        vmaas-qe --secrets-local-dir secrets/sanitized"
                }
            }
            
        }

        stage("Integration tests") {
            // Run integration tests
            withStatusContext.custom("integration-tests") {
                stage("Run vmaas-webapp with coverage") {
                    // Start webapp as `coverage run ...` instead of `python ...` to collect coverage
                    sh '''
                        webapp_pod="$(oc get pods | grep 'Running' | grep 'webapp' | awk '{print $1}')"
                        oc exec "${webapp_pod}" -- bash -c "coverage run webapp/app.py --source webapp &>/proc/1/fd/1" &
                    '''
                }
                stage("Setup DB") {
                    withCredentials([string(credentialsId: "vmaas-bot-token", variable: "TOKEN"),
                                    file(credentialsId: "repolist-json", variable: "REPOLIST")]) {
                    sh """
                        cd vulnerability
                        vmaas/scripts/setup_db.sh ${REPOLIST} \
                            http://vmaas-reposcan.vmaas-qe.svc:8081 \
                            http://vmaas-webapp.vmaas-qe.svc:8080 \
                            ${TOKEN}
                        sleep 10
                    """   
                    }
 
                }
                stage("Run tests") {
                    // Running pytest can result in six different exit codes:
                    // 0: All tests were collected and passed successfully
                    // 1: Tests were collected and run but some of the tests failed
                    // 2: Test execution was interrupted by the user
                    // 3: Internal error happened while executing tests
                    // 4: pytest command line usage error
                    // 5: No tests were collected
                    sh '''
                        pytest_status=0
                        iqe tests plugin vulnerability -vvv -r s -k tests/vmaas --html="report.html" --self-contained-html --generate-report || pytest_status="$?"
                        if [ "$pytest_status" -gt 1 ]; then
                            exit "$pytest_status"
                        fi
                        mkdir html_report
                        mv report.html html_report
                    '''
                }
            }
            junit "iqe-junit-report.xml"
        }

        stage("Code coverage") {
            // Notifies GitHub with the "continuous-integration/jenkins/coverage" status
            // Kill webapp and copy coverage in html format
            webapp_pod = sh(
                script: "oc get pods | grep 'Running' | grep 'webapp' | awk '{print \$1}'",
                returnStdout: true
            ).trim()

            sh "oc exec ${webapp_pod} -- pkill -sigint coverage"
            
            def status = 99
            status = sh(
                script: "oc exec ${webapp_pod} -- coverage html --fail-under=${codecovThreshold} --omit /opt/\\* -d /tmp/htmlcov",
                returnStatus: true
            )

            sh "mkdir htmlcov"
            sh "oc cp ${webapp_pod}:/tmp/htmlcov htmlcov"

            withStatusContext.coverage {
                assert status == 0
            }

            // run webapp's app.py again
            sh "oc exec ${webapp_pod} -- bash -c 'webapp/app.py &>/proc/1/fd/1' &"
        }

        stage("Publish in Polarion") {
            // Publish results for tagged commits in Polarion
            sh '''
                last_commit="$(git rev-parse --short HEAD)"
                versioned="$(git describe --tags --exact-match "$last_commit" 2>/dev/null)" || true
                if [ -n "$versioned" ]; then
                    iqe results plugin vulnerability -t test-run-vmaas-$versioned
                fi
            '''
        }

        stage("Publish HTML") {
            publishHTML (target: [
                  allowMissing: true,
                  alwaysLinkToLastBuild: true,
                  keepAll: true,
                  reportDir: 'htmlcov',
                  reportFiles: 'index.html',
                  reportName: "Coverage Report"
            ])
            publishHTML (target: [
                  allowMissing: true,
                  alwaysLinkToLastBuild: true,
                  keepAll: true,
                  reportDir: 'html_report',
                  reportFiles: 'report.html',
                  reportName: "Test Report"
            ])
        }
    }
}
