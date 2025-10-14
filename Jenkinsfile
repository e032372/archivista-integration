// Jenkinsfile
pipeline {
  agent any

  environment {
    ARCHIVISTA_URL = 'http://archivista:8082'
  }

  stages {
    stage('Install Tools') {
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail
set -x
curl -sSL https://raw.githubusercontent.com/in-toto/witness/main/install-witness.sh -o install-witness.sh
bash install-witness.sh
witness version || true

# Install jq (needed to extract subjects)
if ! command -v jq >/dev/null 2>&1; then
  curl -L -o /usr/local/bin/jq https://github.com/jqlang/jq/releases/download/jq-1.7.1/jq-linux-amd64
  chmod +x /usr/local/bin/jq
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

file="attestations/build.json"

echo "Decoding DSSE payload to extract subjects..."
payload_b64=$(jq -r '.payload' "$file")
echo "$payload_b64" | base64 -d > attestations/build.decoded.json

echo "Subjects found:"
jq -r '.subject[] | "\(.name) \(.digest.sha256)"' attestations/build.decoded.json || true

# Build --subjects argument list: name=digest pairs
SUBJECT_ARGS=$(jq -r '.subject[] | "\(.name)=\(.digest.sha256)"' attestations/build.decoded.json | tr '\\n' ' ')
echo "Using subjects: $SUBJECT_ARGS"

set +e
witness verify "$file" --subjects $SUBJECT_ARGS
rc=$?
set -e
if [ $rc -ne 0 ]; then
  echo "Witness verify failed. Dumping first 400 chars of decoded payload:"
  head -c 400 attestations/build.decoded.json || true
  exit $rc
fi

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

remote="attestations/remote/build-remote.json"
mkdir -p attestations/remote

witness archivista get \
  --archivista-server "${ARCHIVISTA_URL}" \
  --step build \
  --output "${remote}"

echo "Decoding remote DSSE payload..."
payload_b64=$(jq -r '.payload' "$remote")
echo "$payload_b64" | base64 -d > attestations/remote/build-remote.decoded.json

echo "Remote subjects:"
jq -r '.subject[] | "\(.name) \(.digest.sha256)"' attestations/remote/build-remote.decoded.json || true

REMOTE_SUBJECT_ARGS=$(jq -r '.subject[] | "\(.name)=\(.digest.sha256)"' attestations/remote/build-remote.decoded.json | tr '\\n' ' ')
echo "Using subjects: $REMOTE_SUBJECT_ARGS"

set +e
witness verify "$remote" --subjects $REMOTE_SUBJECT_ARGS
rc=$?
set -e
if [ $rc -ne 0 ]; then
  echo "Remote witness verify failed. Snippet:"
  head -c 400 attestations/remote/build-remote.decoded.json || true
  exit $rc
fi

echo "Remote attestation verification succeeded."
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
