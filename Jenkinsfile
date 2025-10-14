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
// groovy
    // groovy
    // Stage replacement for `Jenkinsfile`
// Groovy
stage('Create & Sign Policy') {
  steps {
    sh '''#!/usr/bin/env bash
set -euo pipefail
set -x

ATT_FILE="attestations/build.json"
[ -r "$ATT_FILE" ] || { echo "Missing $ATT_FILE" >&2; exit 1; }

# Base64 encode public key (single line)
if command -v base64 >/dev/null 2>&1; then
  PUBKEY_PEM_B64="$(base64 -w0 testpub.pem 2>/dev/null || base64 < testpub.pem)"
else
  PUBKEY_PEM_B64="$(openssl base64 -A < testpub.pem)"
fi

# Compute KEYID (sha256 of decoded PEM)
KEYID="$(
  printf '%s' "$PUBKEY_PEM_B64" \
  | (base64 -d 2>/dev/null || openssl base64 -d -A 2>/dev/null) \
  | (sha256sum 2>/dev/null || shasum -a 256) \
  | awk '{print $1}'
)"

extract_pred_type() {
  # 1. Direct search in top-level JSON
  local pt
  pt="$(grep -o '"predicateType"[[:space:]]*:[[:space:]]*"[^"]*"' "$ATT_FILE" 2>/dev/null | head -n1 | cut -d'"' -f4 || true)"
  [ -n "$pt" ] && { printf '%s' "$pt"; return 0; }

  # 2. Attempt DSSE payload extraction
  local payload
  payload="$(grep -o '"payload"[[:space:]]*:[[:space:]]*"[^"]*"' "$ATT_FILE" 2>/dev/null | head -n1 | cut -d'"' -f4 || true)"
  [ -z "$payload" ] && return 1

  # Normalize (URL-safe to standard) and pad base64
  payload="${payload//-/+}"
  payload="${payload//_/\/}"
  local mod=$(( ${#payload} % 4 ))
  if [ $mod -ne 0 ]; then
    payload="${payload}$(printf '=%.0s' $(seq 1 $((4-mod))))"
  fi

  # Decode payload to temp file
  if ! printf '%s' "$payload" | base64 -d 2>/dev/null > /tmp/payload_decoded.json; then
    return 1
  fi

  # 3. Search decoded JSON
  pt="$(grep -o '"predicateType"[[:space:]]*:[[:space:]]*"[^"]*"' /tmp/payload_decoded.json 2>/dev/null | head -n1 | cut -d'"' -f4 || true)"
  [ -n "$pt" ] && { printf '%s' "$pt"; return 0; }
  return 1
}

PRED_TYPE="$(extract_pred_type || true)"

if [ -z "$PRED_TYPE" ]; then
  echo "Failed to extract predicateType from $ATT_FILE" >&2
  exit 1
fi

# Write policy.json
cat > policy.json <<POLICY
{
  "expires": "2035-12-17T23:57:40-05:00",
  "steps": {
    "build": {
      "name": "build",
      "attestations": [
        {"type":"https://witness.dev/attestations/material/v0.1"},
        {"type":"https://witness.dev/attestations/product/v0.1"},
        {"type":"https://witness.dev/attestations/command-run/v0.1"}
      ],
      "functionaries": [
        {"type":"publickey","publickeyid":"${KEYID}"}
      ],
      "collections": [
        {
          "name": "dist-files",
          "patterns": ["dist/**"],
          "verifiers": [
            {
              "type": "attestation",
              "attestation_type": "https://witness.dev/attestations/product/v0.1",
              "predicate_type": "${PRED_TYPE}"
            }
          ]
        }
      ]
    }
  },
  "publickeys": {
    "${KEYID}": {
      "keyid": "${KEYID}",
      "key": "${PUBKEY_PEM_B64}"
    }
  }
}
POLICY

# Optional signing
if command -v witness >/dev/null 2>&1; then
  witness sign --signer-file-key-path testkey.pem -f policy.json -o policy-signed.json
else
  echo "witness CLI not found; skipping signing" >&2
fi

stash name: 'policy-out', includes: 'policy*.json'
'''
  }
}




    stage('Verify (policy-based)') {
      when { expression { return fileExists('testpub.pem') } }
      steps {
        unstash 'witness-out'
        unstash 'policy-out'
        sh '''#!/usr/bin/env bash
set -euo pipefail
set -x
test -f dist/app.txt
test -f attestations/build.json
test -f policy-signed.json

witness verify \
  --attestations attestations/build.json \
  -f dist/app.txt \
  -p policy-signed.json \
  -k testpub.pem
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
