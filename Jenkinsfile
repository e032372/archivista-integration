// Jenkinsfile
pipeline {
  agent any

  environment {
    ARCHIVISTA_URL = 'http://archivista:8082'
    // Optionally pin to a known version that includes Archivista support:
    // WITNESS_VERSION = 'v0.3.7'
  }

  stages {

    stage('Install Tools') {
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail
set -x

# Install Witness
if [ -n "${WITNESS_VERSION:-}" ]; then
  echo "Pinning Witness to ${WITNESS_VERSION}"
fi
curl -sSL https://raw.githubusercontent.com/in-toto/witness/main/install-witness.sh -o install-witness.sh
bash install-witness.sh
witness version || true

# Ensure jq is available (install to a local path to avoid needing root)
if ! command -v jq >/dev/null 2>&1; then
  JQ_BIN="${PWD}/.local/bin/jq"
  mkdir -p "$(dirname "$JQ_BIN")"
  curl -L -o "$JQ_BIN" https://github.com/jqlang/jq/releases/download/jq-1.7.1/jq-linux-amd64
  chmod +x "$JQ_BIN"
  export PATH="$(dirname "$JQ_BIN"):$PATH"
fi
jq --version || true
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

# Create the product *inside* the witness-wrapped command so it becomes a recorded subject.
witness run \
  --step build \
  --signer-file-key-path testkey.pem \
  --enable-archivista \
  --archivista-server "${ARCHIVISTA_URL}" \
  -o attestations/build.json -- \
  bash -lc 'echo "hello $(date)" > dist/app.txt && echo "Simulated build complete"'

# For visibility, show the DSSE inner payload and subjects
jq -r '.payload' attestations/build.json | base64 -d | jq . || true
jq -r '.payload' attestations/build.json | base64 -d | jq -r '.subject[] | [.name, .digest.sha256] | @tsv' || true
'''
        stash name: 'witness-out', includes: 'dist/**,attestations/**,testpub.pem,policy*.json'
      }
    }

    stage('Verify Attestation (from Archivista)') {
      steps {
        unstash 'witness-out'
        sh '''#!/usr/bin/env bash
set -euo pipefail
set -x

# Compute the subject hash; prefer using the attestation payload if present, otherwise hash the file.
SUBJECT_HASH=$(jq -r '.payload' attestations/build.json | base64 -d | \
  jq -r '.subject[] | select(.name=="dist/app.txt") | .digest.sha256' || true)

if [ -z "${SUBJECT_HASH}" ]; then
  SUBJECT_HASH=$(sha256sum dist/app.txt | awk '{print $1}')
fi
# Only include policy if present (useful for first-run smoke tests)
POLICY_FLAG=""
if [ -f policy-signed.json ]; then
  POLICY_FLAG="--policy policy-signed.json"
fi
# Let Witness retrieve from Archivista using the subject (no manual GraphQL needed)
# Flags reference: verify supports --enable-archivista, --archivista-server, --subjects, -f/--artifactfile.
witness verify \
  --enable-archivista \
  --archivista-server "${ARCHIVISTA_URL}" \
  --policy policy-signed.json \
  --publickey testpub.pem \
  --subjects "dist/app.txt=${SUBJECT_HASH}" \
  --artifactfile dist/app.txt
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
