pipeline {
  agent any

  environment {
    ARCHIVISTA_URL = 'http://archivista:8082'
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

    stage('Build & Attest (store in Archivista)') {
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
        stash name: 'witness-out', includes: 'dist/**,attestations/**'
      }
    }

    stage('Verify Attestation (local)') {
      steps {
        unstash 'witness-out'
        sh '''#!/usr/bin/env bash
set -euo pipefail
set -x

# Lightweight optional JSON glance (no python dependency)
if command -v jq >/dev/null 2>&1; then
  jq 'keys' attestations/build.json
else
  echo "jq not found; showing first 200 chars:"
  head -c 200 attestations/build.json || true
  echo
fi

witness verify \
  --attestation attestations/build.json \
  --public-key testpub.pem

grep -q "hello" dist/app.txt
echo "Local attestation verification succeeded."
'''
      }
    }

    stage('Verify Attestation (from Archivista)') {
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail
set -x
mkdir -p attestations/remote

witness archivista get \
  --archivista-server "${ARCHIVISTA_URL}" \
  --step build \
  --output attestations/remote/build-remote.json

if command -v jq >/dev/null 2>&1; then
  jq 'keys' attestations/remote/build-remote.json
else
  echo "jq not found; showing first 200 chars:"
  head -c 200 attestations/remote/build-remote.json || true
  echo
fi

witness verify \
  --attestation attestations/remote/build-remote.json \
  --public-key testpub.pem

echo "Remote (Archivista) attestation verification succeeded."
'''
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'attestations/**/*.json, dist/**, policy*.json', fingerprint: true
    }
  }
}
