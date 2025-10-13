pipeline {
  agent any

  environment {
    ARCHIVISTA_URL = 'http://archivista:8082'   // Archivista on the same Docker network
    // For host-process Archivista, use: ARCHIVISTA_URL = 'http://host.docker.internal:8082'
  }

  stages {
    stage('Install Witness CLI') {
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail
set -x
curl -sSL https://raw.githubusercontent.com/in-toto/witness/main/install-witness.sh -o install-witness.sh
bash install-witness.sh
witness version
'''
      }
    }

    stage('Generate Signing Keys (demo)') {
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail
set -x
openssl genpkey -algorithm ed25519 -out testkey.pem
openssl pkey -in testkey.pem -pubout > testpub.pem
'''
      }
    }

    stage('Build & Attest') {
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail
set -x
mkdir -p dist attestations
echo "hello $(date)" > dist/app.txt

witness run \
  --step build \
  --signer-file-key-path testkey.pem \
  --enable-archivista \
  --archivista-server "${ARCHIVISTA_URL}" \
  -o attestations/build.json -- \
  bash -lc 'echo "Simulated build complete"'
'''
        // Jenkins step (not shell): move files across stages
        stash name: 'witness-out', includes: 'dist/**,attestations/**'
      }
    }

    stage('Verify (optional policy)') {
      when { expression { return fileExists('testpub.pem') } }
      steps {
        unstash 'witness-out'
        sh '''#!/usr/bin/env bash
set -euo pipefail
set -x
test -f dist/app.txt
test -f attestations/build.json

# Local verification using our public key as trust source
witness verify \
  --attestations attestations/build.json \
  -f dist/app.txt \
  -k testpub.pem
'''
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'attestations/**/*.json, dist/**', fingerprint: true
    }
  }
}
