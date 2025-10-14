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
ATT=attestations/build.json
DEC=attestations/build.decoded.json

jq -r '.payload' "$ATT" | base64 -d > "$DEC"

echo "Subjects (name sha256):"
jq -r '.subject[] | [.name, .digest.sha256] | @tsv' "$DEC" || true

# Build subject flags safely
SUBJECT_FLAGS=""
while IFS=$'\\t' read -r name digest; do
  [ -z "${name:-}" ] && continue
  [ -z "${digest:-}" ] && continue
  SUBJECT_FLAGS+=" --subjects ${name}=${digest}"
done < <(jq -r '.subject[] | [.name, .digest.sha256] | @tsv' "$DEC")

echo "Using subject flags:${SUBJECT_FLAGS}"

set +e
# Use correct flag: --publickey
witness verify "$ATT" --publickey testpub.pem ${SUBJECT_FLAGS}
rc=$?
set -e
if [ $rc -ne 0 ]; then
  echo "Witness verify failed. Decoded payload snippet:"
  head -c 400 "$DEC" || true
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
REMOTE=attestations/remote/build-remote.json
DEC_REMOTE=attestations/remote/build-remote.decoded.json
mkdir -p attestations/remote

witness archivista get \
  --archivista-server "${ARCHIVISTA_URL}" \
  --step build \
  --output "${REMOTE}"

jq -r '.payload' "$REMOTE" | base64 -d > "$DEC_REMOTE"

echo "Remote subjects (name sha256):"
jq -r '.subject[] | [.name, .digest.sha256] | @tsv' "$DEC_REMOTE" || true

REMOTE_SUBJECT_FLAGS=""
while IFS=$'\\t' read -r name digest; do
  [ -z "${name:-}" ] && continue
  [ -z "${digest:-}" ] && continue
  REMOTE_SUBJECT_FLAGS+=" --subjects ${name}=${digest}"
done < <(jq -r '.subject[] | [.name, .digest.sha256] | @tsv' "$DEC_REMOTE")

echo "Using remote subject flags:${REMOTE_SUBJECT_FLAGS}"

set +e
witness verify "$REMOTE" --publickey testpub.pem ${REMOTE_SUBJECT_FLAGS}
rc=$?
set -e
if [ $rc -ne 0 ]; then
  echo "Remote witness verify failed. Decoded payload snippet:"
  head -c 400 "$DEC_REMOTE" || true
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
