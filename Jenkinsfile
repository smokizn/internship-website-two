// Deploy-Website-Two — declarative pipeline, loaded via "Pipeline script from SCM".
// The job definition owns the repository URL; this file only uses the checked-out workspace.
pipeline {
    agent any

    options {
        skipDefaultCheckout(true)          // single explicit checkout in the Checkout stage
        disableConcurrentBuilds()
        timestamps()
        timeout(time: 10, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    environment {
        SITE_NAME          = 'website-two'
        SITE_PORT          = '8092'
        DEPLOY_CREDENTIALS = 'deploy-website-two'   // SSH key locked to this site by a forced command
        DEPLOY_HOST        = 'internship-deployer'
        DEPLOY_PORT        = '2222'
        WEB_HOST           = 'internship-apache'
        KNOWN_HOSTS        = '/run/secrets/deployer_known_hosts'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git log -1 --format="Commit %h by %an: %s"'
            }
        }

        stage('Validate') {
            steps {
                sh '''#!/bin/bash
                    set -euo pipefail
                    for f in index.html style.css; do
                        if [ ! -s "$f" ]; then
                            echo "FAIL: $f is missing or empty"; exit 1
                        fi
                        echo "OK: $f ($(wc -c < "$f") bytes)"
                    done
                    grep -q 'href="style.css"' index.html || { echo "FAIL: index.html does not link style.css"; exit 1; }

                    # tidy exit codes: 0 = clean, 1 = warnings only, 2 = errors
                    rc=0; tidy -q -e --show-warnings yes --drop-empty-elements no index.html || rc=$?
                    if [ "$rc" -ge 2 ]; then echo "FAIL: HTML errors found"; exit 1; fi
                    echo "OK: HTML syntax check passed (tidy rc=$rc)"
                '''
            }
        }

        stage('Deploy') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: env.DEPLOY_CREDENTIALS,
                                                   keyFileVariable: 'DEPLOY_KEY',
                                                   usernameVariable: 'DEPLOY_USER')]) {
                    sh '''#!/bin/bash
                        set -euo pipefail
                        printf 'site=%s\\ncommit=%s\\nbuild=%s\\n' \
                            "$SITE_NAME" "$(git rev-parse HEAD)" "$BUILD_NUMBER" > build-info.txt
                        # Only the site files are shipped; the deploy endpoint chooses the target directory.
                        tar --owner=0 --group=0 -czf - index.html style.css build-info.txt \
                          | ssh -T -i "$DEPLOY_KEY" -p "$DEPLOY_PORT" \
                                -o IdentitiesOnly=yes -o BatchMode=yes \
                                -o StrictHostKeyChecking=yes -o UserKnownHostsFile="$KNOWN_HOSTS" \
                                "$DEPLOY_USER@$DEPLOY_HOST"
                    '''
                }
            }
        }

        stage('Verify') {
            steps {
                sh '''#!/bin/bash
                    set -euo pipefail
                    url="http://$WEB_HOST:$SITE_PORT"
                    curl -fsS -o served.html "$url/"
                    cmp -s served.html index.html || { echo "FAIL: served page differs from commit"; exit 1; }
                    curl -fsS "$url/build-info.txt"
                    echo "OK: $SITE_NAME is serving commit $(git rev-parse --short HEAD) on port $SITE_PORT"
                '''
            }
        }
    }

    post {
        success {
            echo "SUCCESS: ${env.SITE_NAME} build #${env.BUILD_NUMBER} deployed and verified on port ${env.SITE_PORT}."
        }
        failure {
            echo "FAILURE: ${env.SITE_NAME} build #${env.BUILD_NUMBER} failed. The previous release is still live."
        }
        always {
            sh 'rm -f build-info.txt served.html'
        }
    }
}
