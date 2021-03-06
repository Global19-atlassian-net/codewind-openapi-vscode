#!groovy​

import java.text.SimpleDateFormat;

@NonCPS
def getChangeDetail() {

    def changeDetail = ""
    def changes = currentBuild.changeSets
    
    for (int i = 0; i < changes.size(); i++) {
        
        def entries = changes[i].items
    
        for (int j = 0; j < entries.length; j++) {
            def entry = entries[j]

            Date entry_timestamp = new Date(entry.timestamp)
            SimpleDateFormat timestamp = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
            changeDetail += "${entry.commitId.take(7)} by [${entry.author}] on ${timestamp.format(entry_timestamp)}: ${entry.msg}\n\n"
        }
    }

    if (!changeDetail) {
        changeDetail = "No new changes"
    }
    return changeDetail
}

IS_MASTER_BRANCH = env.BRANCH_NAME == "master"
IS_RELEASE_BRANCH = (env.BRANCH_NAME ==~ /\d+\.\d+\.\d+/)

def sendEmailNotification() {    
    if (!(IS_MASTER_BRANCH || IS_RELEASE_BRANCH)) {
        println "Skipping email step since branch condition was not met"
        return
    }

    final def EXTRECIPIENTS = emailextrecipients([
        [$class: 'CulpritsRecipientProvider'],
        [$class: 'RequesterRecipientProvider'],
        [$class: 'DevelopersRecipientProvider']
    ])

    def RECIPIENTS = EXTRECIPIENTS != null ? EXTRECIPIENTS : env.CHANGE_AUTHOR_EMAIL;
    def SUBJECT = "${env.JOB_NAME} - Build #${env.BUILD_NUMBER} - ${currentBuild.currentResult}!"
    def CHANGE = getChangeDetail();
    
    emailext(
        to: RECIPIENTS,
        subject: "${SUBJECT}",
        body: """
            <b>Build result:</b> 
            <br><br>
            ${currentBuild.currentResult}
            <br><br>
            <b>Project name - Build number:</b>
            <br><br>
            ${currentBuild.fullProjectName} - Build #${env.BUILD_NUMBER}
            <br><br>
            <b>Check console output for more detail:</b> 
            <br><br>
            <a href="${currentBuild.absoluteUrl}console">${currentBuild.absoluteUrl}console</a>
            <br><br>
            <b>Changes:</b> 
            <br><br>
            ${CHANGE}
            <br><br>
        """
    );
}

pipeline {
    agent {
        kubernetes {
              label 'node'
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: node
    image: node:lts
    tty: true
    command:
      - cat
"""
        }
    }

    options {
        timestamps()
        skipStagesAfterUnstable()
    }

    environment {
        // https://stackoverflow.com/a/43264045
        HOME="."
    }

    stages {
        stage('Build') {
            steps {
                container("node") {
                    dir('dev') {
                        sh '''#!/usr/bin/env bash
                            # Test compilation to catch any errors
                            npm run vscode:prepublish

                            # Package for prod
                            npm i vsce
                            npx vsce package

                            # rename to have datetime for clarity + prevent collisions
                            export artifact_name=$(basename *.vsix)
                            mv -v $artifact_name ${artifact_name/.vsix/_$(date +'%F-%H%M').vsix}
                        '''

                        // Note there must be exactly one .vsix
                        stash includes: '*.vsix', name: 'deployables'
                    }
                }
            }
        }
        stage ('Deploy') {
            // This when clause disables PR build uploads; you may comment this out if you want your build uploaded.
            when {
                beforeAgent true
                not {
                    changeRequest()
                }
            }

            options {
                skipDefaultCheckout()
            }

            agent any
            steps {
                sshagent (['projects-storage.eclipse.org-bot-ssh']) {
                    unstash 'deployables'
                    
                    println("Deploying codewind-openapi-vscode to archive area...")

                    sh '''#!/usr/bin/env bash
                        export REPO_NAME="codewind-openapi-vscode"
                        export OUTPUT_NAME="codewind-openapi-tools"
                        export ARCHIVE_AREA_URL="https://archive.eclipse.org/codewind/$REPO_NAME"
                        export LATEST_DIR="latest"
                        export BUILD_INFO="build_info.properties"
                        export sshHost="genie.codewind@projects-storage.eclipse.org"
                        export deployDir="/home/data/httpd/archive.eclipse.org/codewind/$REPO_NAME"
                        
                        if [ -z $CHANGE_ID ]; then
                            UPLOAD_DIR="$GIT_BRANCH/$BUILD_ID"
                            BUILD_URL="$ARCHIVE_AREA_URL/$UPLOAD_DIR"

                            ssh $sshHost rm -rf $deployDir/$GIT_BRANCH/$LATEST_DIR
                            ssh $sshHost mkdir -p $deployDir/$GIT_BRANCH/$LATEST_DIR

                            cp $OUTPUT_NAME-*.vsix $OUTPUT_NAME.vsix
                            scp $OUTPUT_NAME.vsix $sshHost:$deployDir/$GIT_BRANCH/$LATEST_DIR/$OUTPUT_NAME.vsix

                            echo "# Build date: $(date +%F-%T)" >> $BUILD_INFO
                            echo "build_info.url=$BUILD_URL" >> $BUILD_INFO
                            SHA1=$(sha1sum ${OUTPUT_NAME}.vsix | cut -d ' ' -f 1)
                            echo "build_info.SHA-1=${SHA1}" >> $BUILD_INFO

                            scp $BUILD_INFO $sshHost:$deployDir/$GIT_BRANCH/$LATEST_DIR/$BUILD_INFO
                            rm $BUILD_INFO
                            rm $OUTPUT_NAME.vsix
                        else
                            UPLOAD_DIR="pr/$CHANGE_ID/$BUILD_ID"
                        fi

                        ssh $sshHost rm -rf $deployDir/${UPLOAD_DIR}
                        ssh $sshHost mkdir -p $deployDir/${UPLOAD_DIR}
                        ls *.vsix
                        scp -r *.vsix $sshHost:$deployDir/${UPLOAD_DIR}

                        echo "Uploaded to https://archive.eclipse.org${deployDir##*archive.eclipse.org}"
                    '''
                }
            }
        }
    }

    post {
        failure {
            sendEmailNotification()
        }
    }
}
