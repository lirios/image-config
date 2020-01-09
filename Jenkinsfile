pipeline {
  parameters {
    choice(name:'type', choices:['live', 'disk'], description:'Image type')
  }
  agent {
    docker {
      image "liridev/ci-fedora-jenkins:31"
      args "--privileged -v ${JENKINS_HOME}:${JENKINS_HOME} --tmpfs /tmp -v /var/tmp:/var/tmp --device /dev/fuse"
      alwaysPull true
    }
  }
  environment {
      IMAGE_MANAGER_CREDENTIALS = credentials('image-manager')
  }
  stages {
    stage('Prepare') {
      steps {
        sh """
sudo dnf install -y dnf-plugins-core gnupg2 pinentry python3-requests python3-requests-toolbelt
sudo dnf copr enable -y plfiorini/liri-tools
sudo dnf install -y ostree-image-creator
"""
        withCredentials([file(credentialsId: 'ci-pgp-key', variable: 'FILE')]) {
          sh label: 'Import PGP key', script: "gpg --import --no-tty --batch --yes ${FILE}"
        }
      }
    }
    stage('Create') {
      steps {
        script {
          def now = new Date()
          today = now.format("yyyyMMdd", TimeZone.getTimeZone('Europe/Rome'))
          imageName = "lirios-${today}-x86_64"
          isoFileName = "${imageName}.iso"
          checksumFileName = "${imageName}-CHECKSUM"
          workDir = "${JENKINS_HOME}/workspace/ostree-artifacts/work"
          repoPath = "${JENKINS_HOME}/workspace/ostree-artifacts/repo-dev"
        }
        sh label: 'Create image', script: "sudo RUST_BACKTRACE=1 RUST_LOG=trace oic --manifest=${params.type}.yaml --output=${isoFileName} --fslabel=${imageName} --workdir=${workDir} --repo=${repoPath}"
        withCredentials([file(credentialsId: 'ci-pgp-passphrase', variable: 'FILE')]) {
          sh label: 'Checksum', script: """
mv ${checksumFileName} ${checksumFileName}.orig
cat ${checksumFileName}.orig | gpg --clearsign --pinentry-mode=loopback --passphrase-file=${FILE} --no-tty --batch --yes > ${checksumFileName}
"""
        }
      }
    }
    stage('Publish') {
      steps {
        sh "curl -O https://raw.githubusercontent.com/liri-infra/image-manager/develop/image-manager-client && chmod 755 image-manager-client"
        script {
          token = sh(returnStdout: true, script: "echo ${env.IMAGE_MANAGER_CREDENTIALS_PSW} | ./image-manager-client create-token --api-url=${env.IMAGE_MANAGER_URL} ${env.IMAGE_MANAGER_CREDENTIALS_USR}").trim()
        }
        sh "./image-manager-client upload --api-url=${env.IMAGE_MANAGER_URL} --token=${token} --channel=nightly --image=${isoFileName} --checksum=${checksumFileName}"
        sh "sudo rm -f ${isoFileName} ${checksumFileName} image-manager-client"
      }
    }
  }
}
