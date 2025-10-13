// Jenkinsfile
// Pipeline: Witness attestation → store in Archivista → policy-based verification

pipeline {
  agent any

  environment {
    // Archivista reachable on the same Docker host/network
    ARCHIVISTA_URL = 'http://archivista:8082'
    // If Archivista runs natively on the host, you can use:
    // ARCHIVISTA_URL = 'http://host.docker.internal:8082'
  }

  stages {

    stage('Install Witness CLI') {
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail
set -x
# Install Witness from official script
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
# Demo Ed25519 keypair for signing attestations & policy
# For production, prefer Sigstore keyless or a KMS (see Witness docs).
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

# Create an attestation and upload to Archivista
witness run \
  --step build \
  --signer-file-key-path testkey.pem \
  --enable-archivista \
  --archivista-server "${ARCHIVISTA_URL}" \
  -o attestations/build.json -- \
  bash -lc 'echo "Simulated build complete"'
'''
        // Make outputs available to later stages (agnostic of node/container changes)
        stash name: 'witness-out', includes: 'dist/**,attestations/**'
      }
    }

    stage('Create & Sign Policy') {
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail
set -x
# Compute the functionary KeyID exactly as Witness does: SHA-256 of the DER SubjectPublicKeyInfo
KEYID="$(openssl pkey -pubin -in testpub.pem -outform DER 2>/dev/null | sha256sum | awk '{print $1}')"

# Embed the public key as base64 of the *PEM text* (NOT DER).
# When verify decodes it, it must see a valid PEM block.
PUBKEY_PEM_B64="$(base64 -w0 testpub.pem 2>/dev/null || openssl base64 -A < testpub.pem)"

# Minimal policy: single 'build' step requiring material/product/command-run attestations
# and trusting the functionary identified by KEYID.
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

# Inject the real KEYID and base64 PEM public key
sed -i "s|KEYID_PLACEHOLDER|${KEYID}|g" policy.json
sed -i "s|PUBKEY_BASE64_PLACEHOLDER|${PUBKEY_PEM_B64}|g" policy.json

# Sign the policy into a DSSE envelope (policy-signed.json)
witness sign \
  --signer-file-key-path testkey.pem \
  -f policy.json \
  -o policy-signed.json
'''
        // Stash the signed policy as well
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
# Sanity checks
test -f dist/app.txt
test -f attestations/build.json
test -f policy-signed.json

# Verify attestations against the signed policy using the policy's public key for trust
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
