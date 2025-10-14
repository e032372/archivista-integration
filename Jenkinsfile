// groovy
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
stage('Create & Sign Policy') {
  steps {
    sh '''#!/usr/bin/env bash
set -euo pipefail
set -x

PUBKEY_PEM_B64="$(base64 -w0 testpub.pem 2>/dev/null || openssl base64 -A < testpub.pem)"
KEYID="$(printf '%s' "${PUBKEY_PEM_B64}" | base64 -d | sha256sum | awk '{print $1}')"

# ensure file exists and is readable
if [ ! -r "attestations/build.json" ]; then
  echo "attestations/build.json missing or not readable" >&2
  ls -l attestations || true
  exit 1
fi

# extract predicateType (portable grep+cut)
PRED_TYPE="$(grep -o '"predicateType"[[:space:]]*:[[:space:]]*"[^"]*"' attestations/build.json 2>/dev/null | head -n1 | cut -d\" -f4 || true)"

# fallback: extract DSSE payload and decode then search there
if [ -z "${PRED_TYPE}" ]; then
  PAYLOAD_B64="$(grep -o '"payload"[[:space:]]*:[[:space:]]*"[^"]*"' attestations/build.json 2>/dev/null | head -n1 | cut -d\" -f4 || true)"
  if [ -n "${PAYLOAD_B64}" ]; then
    PRED_TYPE="$(printf '%s' "${PAYLOAD_B64}" | base64 -d 2>/dev/null | grep -o '"predicateType"[[:space:]]*:[[:space:]]*"[^"]*"' | head -n1 | cut -d\" -f4 || true)"
  fi
fi

if [ -z "${PRED_TYPE}" ]; then
  echo "Failed to extract predicateType from `attestations/build.json`" >&2
  exit 1
fi

cat > policy.json <<'POLICY'
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
        {"type":"publickey","publickeyid":"KEYID_PLACEHOLDER"}
      ],
      "collections": [
        {
          "name": "dist-files",
          "patterns": ["dist/**"],
          "verifiers": [
            {
              "type": "attestation",
              "attestation_type": "https://witness.dev/attestations/product/v0.1",
              "predicate_type": "PRED_PLACEHOLDER"
            }
          ]
        }
      ]
    }
  },
  "publickeys": {
    "KEYID_PLACEHOLDER": {
      "keyid": "KEYID_PLACEHOLDER",
      "key": "PUBKEY_BASE64_PLACEHOLDER"
    }
  }
}
POLICY

# substitute placeholders
sed -i "s|KEYID_PLACEHOLDER|${KEYID}|g" policy.json
sed -i "s|PUBKEY_BASE64_PLACEHOLDER|${PUBKEY_PEM_B64}|g" policy.json
sed -i "s|PRED_PLACEHOLDER|${PRED_TYPE}|g" policy.json

# sign the policy
witness sign --signer-file-key-path testkey.pem -f policy.json -o policy-signed.json
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
