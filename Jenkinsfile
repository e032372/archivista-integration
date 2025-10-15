// Jenkinsfile
pipeline {
  agent any

  environment {
    ARCHIVISTA_URL = 'http://archivista:8082'
    // Optional: set WITNESS_VERSION to pin a version that includes archivist/archivista
    // WITNESS_VERSION = 'v0.3.7'
  }

  stages {

    stage('Install Tools') {
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail
set -x
if [ -n "${WITNESS_VERSION:-}" ]; then
  echo "Pinning Witness to ${WITNESS_VERSION}"
fi
curl -sSL https://raw.githubusercontent.com/in-toto/witness/main/install-witness.sh -o install-witness.sh
bash install-witness.sh
witness version || true

if ! command -v jq >/dev/null 2>&1; then
  curl -L -o /usr/local/bin/jq https://github.com/jqlang/jq/releases/download/jq-1.7.1/jq-linux-amd64
  chmod +x /usr/local/bin/jq
fi
jq --version || true

echo "Detecting archivista support..."
HELP_OUT=$(witness run --help 2>&1 || true)
echo "$HELP_OUT" | head -n 50

if echo "$HELP_OUT" | grep -q -- '--archivista-server'; then
  echo archivista > .witness_archi_subcmd
  echo "Using support: archivista"
else
  echo none > .witness_archi_subcmd
  echo "No archivista support available; remote retrieval will be skipped."
fi
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
        stash name: 'witness-out', includes: 'dist/**,attestations/**,.witness_archi_subcmd'
      }
    }

    stage('Verify Attestation (from Archivista)') {
  steps {
    unstash 'witness-out'
    sh '''#!/usr/bin/env bash
set -euo pipefail
set -x

STEP_NAME="build"

# Verify attestation from Archivista
witness verify \
  --archivista-server "${ARCHIVISTA_URL}" \
  --step "${STEP_NAME}" \
  --publickey testpub.pem
'''
  }
}

  }

  post {
    always {
      archiveArtifacts artifacts: 'attestations/**/*.json, dist/**, policy*.json,.witness_archi_subcmd', fingerprint: true
    }
  }
}
