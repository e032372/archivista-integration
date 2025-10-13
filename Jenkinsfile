pipeline {
  agent any

  environment {
    // If Archivista is a container on the same Docker network:
    ARCHIVISTA_URL = 'http://archivista:8082'
    // If Archivista is a host process, use:
    // ARCHIVISTA_URL = 'http://host.docker.internal:8082'
  }

  stages {
    stage('Install Witness CLI') {
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail
set -x
# Install Witness (per official docs) without process substitution
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
# Demo-only Ed25519 keypair; for production use KMS/Sigstore keyless
openssl genpkey -algorithm ed25519 -out testkey.pem
openssl pkey -in testkey.pem -pubout > testpub.pem
'''
      }
    }

    stage('Build & Attest') {
      steps {
        // Do the build and create the attestation
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
        // Now stash the outputs (Jenkins step, not a shell command)
        stash name: 'witness-out', includes: 'dist/**,attestations/**'
      }
    }


stage('Verify (optional policy)') {
  when { expression { return fileExists('testpub.pem') } }
  steps {
    // Bring back build outputs (safe across agents/containers)
    unstash 'witness-out'

    sh '''#!/usr/bin/env bash
set -euo pipefail
set -x

# Sanity checks
test -f dist/app.txt
test -f attestations/build.json

# Verify using local artifact + local attestation (no policy, no Archivista)
witness verify \
  --attestations attestations/build.json \
  -f dist/app.txt
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
