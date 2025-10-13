pipeline {
  agent any

  environment {
    // If Archivista is a container on the same Docker network:
    ARCHIVISTA_URL = 'http://archivista:8082'
    // If Archivista runs on the host, use:
    // ARCHIVISTA_URL = 'http://host.docker.internal:8082'
  }

  stages {
    stage('Install Witness CLI') {
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail
set -x
# Install Witness (per official docs). Avoid process substitution complexity by downloading first.
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
# Demo-only Ed25519 keypair for signing attestations.
# For real pipelines, prefer KMS or Sigstore keyless.
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

# Example "build" – replace with your real build command
mkdir -p dist attestations
echo "hello $(date)" > dist/app.txt

# Create an attestation and upload to Archivista
# --enable-archivista + --archivista-server are the documented flags
witness run \
  --step build \
  --signer-file-key-path testkey.pem \
  --enable-archivista \
  --archivista-server "${ARCHIVISTA_URL}" \
  -o attestations/build.json -- \
  bash -lc 'echo "Simulated build complete"'

# Make outputs available to later stages even if they run on a different executor/container
stash name: 'witness-out', includes: 'dist/**,attestations/**'
'''
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

# Simplest: verify by file path (no digest required)
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
