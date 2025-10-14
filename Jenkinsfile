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

    stage('Create & Sign Policy') {
  steps {
    sh '''#!/usr/bin/env bash
set -euo pipefail
set -x

# portable base64 -> one-line PEM
if command -v base64 >/dev/null 2>&1; then
  PUBKEY_PEM_B64="$(base64 -w0 testpub.pem 2>/dev/null || base64 < testpub.pem)"
else
  PUBKEY_PEM_B64="$(openssl base64 -A < testpub.pem)"
fi

# compute KEYID (sha256 of decoded PEM)
KEYID="$(printf '%s' "${PUBKEY_PEM_B64}" | (base64 -d 2>/dev/null || openssl base64 -d 2>/dev/null) | (sha256sum 2>/dev/null || shasum -a 256) | awk '{print $1}')"

ATT_FILE="attestations/build.json"
if [ ! -r "${ATT_FILE}" ]; then
  echo "attestations/build.json missing or not readable" >&2
  ls -l attestations || true
  exit 1
fi

# try Python JSON traversal first (robust), fall back to grep if Python not present
PRED_TYPE=""
if command -v python3 >/dev/null 2>&1; then
  PRED_TYPE="$(python3 - <<'PY' 2>/dev/null
import json,sys
try:
    data = json.load(open('attestations/build.json'))
except Exception:
    sys.exit(0)
def walk(o):
    if isinstance(o, dict):
        if 'predicateType' in o:
            print(o['predicateType'])
            return True
        for v in o.values():
            if walk(v):
                return True
    elif isinstance(o, list):
        for i in o:
            if walk(i):
                return True
    return False
walk(data)
PY
)"
fi

if [ -z "${PRED_TYPE}" ]; then
  PRED_TYPE="$(grep -o '"predicateType"[[:space:]]*:[[:space:]]*\"[^\"]*\"' "${ATT_FILE}" 2>/dev/null | head -n1 | cut -d\" -f4 || true)"
fi

if [ -z "${PRED_TYPE}" ]; then
  echo "Failed to extract predicateType from ${ATT_FILE}" >&2
  exit 1
fi

# write policy.json using variable expansion (avoids sed portability issues)
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

# sign policy if witness available
if command -v witness >/dev/null 2>&1; then
  witness sign --signer-file-key-path testkey.pem -f policy.json -o policy-signed.json
else
  echo "witness CLI not found; skipping sign step" >&2
fi
'''
    stash name: 'policy-out', includes: 'policy*.json'
  }
}


  post {
    always {
      archiveArtifacts artifacts: 'attestations/**/*.json, dist/**, policy*.json', fingerprint: true
    }
  }
}
