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
# Ensure local jq (if installed) is on PATH for this step too
export PATH="${PWD}/.local/bin:${PATH}"

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
        stash name: 'witness-out', includes: 'dist/**,attestations/**,testpub.pem'
      }
    }

    stage('Verify Attestation (from Archivista)') {
      steps {
        unstash 'witness-out'
        sh '''#!/usr/bin/env bash
set -euo pipefail
set -x

# Ensure local jq (if installed) is on PATH for this step too
export PATH="${PWD}/.local/bin:${PATH}"

# Compute the subject hash; prefer using the attestation payload if present, otherwise hash the file.
SUBJECT_HASH=$(jq -r '.payload' attestations/build.json | base64 -d | \
  jq -r '.subject[] | select(.name=="dist/app.txt") | .digest.sha256' || true)

if [ -z "${SUBJECT_HASH}" ]; then
  SUBJECT_HASH=$(sha256sum dist/app.txt | awk '{print $1}')
fi

# --- Build a minimal permissive policy that trusts the attestation signer (testkey.pem) ---

# 1) Compute keyid as sha256 over DER SPKI
openssl pkey -pubin -in testpub.pem -outform DER > testpub.der
PUB_KEY_ID=$(sha256sum testpub.der | awk '{print $1}')

# 2) Encode the PEM text as base64 for policy.publickeys[].key (NOT DER)
PUB_KEY_PEM_B64=$(base64 testpub.pem | tr -d '\\n')

# (Optional sanity check: ensure the decode starts with a PEM header)
printf '%s' "$PUB_KEY_PEM_B64" | base64 -d | head -1 || true

cat > policy.json <<POLICY
{
  "expires": "2035-01-01T00:00:00Z",
  "publickeys": {
    "${PUB_KEY_ID}": {
      "keyid": "${PUB_KEY_ID}",
      "key": "${PUB_KEY_PEM_B64}"
    }
  },
  "steps": {
    "build": {
      "name": "build",
      "functionaries": [
        { "type": "publickey", "publickeyid": "${PUB_KEY_ID}" }
      ],
      "attestations": [
        { "type": "https://witness.dev/attestations/material/v0.1", "regopolicies": [] },
        { "type": "https://witness.dev/attestations/product/v0.1",  "regopolicies": [] }
      ]
    }
  }
}
POLICY

# Sign the policy with the same key used to sign the attestation collection
witness sign -f policy.json -o policy-signed.json -k testkey.pem

# Verify: pull attestations from Archivista using the subject and enforce the policy
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
