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
witness version || true
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

echo "Attempting witness verify with multiple flag variants (local)..."

attempt_verify () {
  local file="$1"
  local pub="$2"
  local success=""
  # Candidate command forms for differing CLI versions
  cmds=(
    "witness verify ${file} --public-key ${pub}"
    "witness verify --public-key ${pub} ${file}"
    "witness verify ${file} --key ${pub}"
    "witness verify --key ${pub} ${file}"
    "witness verify ${file} --key-file ${pub}"
    "witness verify --key-file ${pub} ${file}"
    "witness verify ${file} ${pub}"
    "witness verify ${file}"
  )
  for c in "${cmds[@]}"; do
    echo "TRY: $c"
    if eval "$c"; then
      echo "SUCCESS with: $c"
      success=1
      break
    fi
  done
  if [ -z "${success:-}" ]; then
    echo "All verify attempts failed. Dumping envelope snippet for debugging:"
    head -c 400 "${file}" || true
    return 1
  fi
}

# Optional brief look (no jq on agent)
echo "Envelope first 200 chars:"
head -c 200 attestations/build.json || true
echo

attempt_verify "attestations/build.json" "testpub.pem"

grep -q "hello" dist/app.txt
echo "Local attestation verification completed."
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

echo "Attempting witness verify with multiple flag variants (remote)..."

attempt_verify () {
  local file="$1"
  local pub="$2"
  local success=""
  cmds=(
    "witness verify ${file} --public-key ${pub}"
    "witness verify --public-key ${pub} ${file}"
    "witness verify ${file} --key ${pub}"
    "witness verify --key ${pub} ${file}"
    "witness verify ${file} --key-file ${pub}"
    "witness verify --key-file ${pub} ${file}"
    "witness verify ${file} ${pub}"
    "witness verify ${file}"
  )
  for c in "${cmds[@]}"; do
    echo "TRY: $c"
    if eval "$c"; then
      echo "SUCCESS with: $c"
      success=1
      break
    fi
  done
  if [ -z "${success:-}" ]; then
    echo "All remote verify attempts failed. Dumping envelope snippet:"
    head -c 400 "${file}" || true
    return 1
  fi
}

echo "Remote envelope first 200 chars:"
head -c 200 attestations/remote/build-remote.json || true
echo

attempt_verify "attestations/remote/build-remote.json" "testpub.pem"

echo "Remote attestation verification completed."
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
