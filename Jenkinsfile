pipeline {
  agent any

  environment {
    ARCHIVISTA_URL = 'http://archivista:8082'   // Archivista on the same Docker network
    // ARCHIVISTA_URL = 'http://host.docker.internal:8082'  // if Archivista runs on the host
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
# Demo keypair for attestations/policy signing (ed25519)
openssl genpkey -algorithm ed25519 -out testkey.pem
openssl pkey -in testkey.pem -pubout > testpub.pem
'''
      }
    }

    stage('Build & Attest') {
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail
set -x
mkdir -p dist attestations
echo "hello $(date)" > dist/app.txt

# Create attestation and upload to Archivista
witness run \
  --step build \
  --signer-file-key-path testkey.pem \
  --enable-archivista \
  --archivista-server "${ARCHIVISTA_URL}" \
  -o attestations/build.json -- \
  bash -lc 'echo "Simulated build complete"'
'''
        // Stash build outputs for later stages
        stash name: 'witness-out', includes: 'dist/**,attestations/**'
      }
    }

    // >>> NEW: create a minimal policy and sign it
    stage('Create & Sign Policy') {
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail
set -x

# Prepare a minimal policy JSON (no external tools). Fields mirror the tutorial's policy template:
# - single 'build' step
# - expects material/product/command-run attestations
# - trusts our public key by keyid, with base64 public key in the publickeys map
KEYID="$(sha256sum testpub.pem | awk '{print $1}')"              # policy publickeyid
PUBKEY_B64="$(openssl base64 -A < testpub.pem)"                  # base64 of PEM

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

# Inject our real KEYID and base64 public key
sed -i "s|KEYID_PLACEHOLDER|${KEYID}|g" policy.json
sed -i "s|PUBKEY_BASE64_PLACEHOLDER|${PUBKEY_B64}|g" policy.json

# Sign the policy into DSSE envelope (policy-signed.json)
witness sign \
  --signer-file-key-path testkey.pem \
  -f policy.json \
  -o policy-signed.json
'''
        // Stash the signed policy too
        stash name: 'policy-out', includes: 'policy*.json'
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

# Verify attestations against the signed policy with the policy's public key
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
