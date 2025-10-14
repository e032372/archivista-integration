// Jenkinsfile
pipeline {
  agent any

  environment {
    ARCHIVISTA_URL = 'http://archivista:8082'
    WITNESS_VERSION = 'v0.3.7'      // adjust to a release that contains archivist/archivista
  }

  stages {

    stage('Install Tools') {
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail
set -x

echo "Installing Witness ${WITNESS_VERSION}"
curl -sSL https://raw.githubusercontent.com/in-toto/witness/main/install-witness.sh -o install-witness.sh
chmod +x install-witness.sh
# Script honors WITNESS_VERSION if exported
export WITNESS_VERSION
bash install-witness.sh || true

echo "Witness after script:"
witness version || true

# Fallback: build from source (ensures latest tagged version with all subcommands)
if ! witness help 2>&1 | grep -E -q '\\barchivist\\b|\\barchivista\\b'; then
  echo "Rebuilding witness from source (fallback)..."
  if ! command -v go >/dev/null 2>&1; then
    echo "Installing minimal Go toolchain"
    curl -sSL https://go.dev/dl/go1.22.5.linux-amd64.tar.gz -o go.tgz
    rm -rf /usr/local/go
    tar -C /usr/local -xzf go.tgz
    export PATH=/usr/local/go/bin:$PATH
  fi
  go install github.com/in-toto/witness/cmd/witness@${WITNESS_VERSION}
  command -v witness
  witness version || true
fi

# Final detection
HELP_OUT=$(witness help 2>&1 || true)
if echo "$HELP_OUT" | grep -E -q '\\barchivist\\b'; then
  echo archivist > .witness_archi_subcmd
elif echo "$HELP_OUT" | grep -E -q '\\barchivista\\b'; then
  echo archivista > .witness_archi_subcmd
else
  echo none > .witness_archi_subcmd
  echo "ERROR: No archivist/archivista subcommand even after rebuild."
fi

# jq
if ! command -v jq >/dev/null 2>&1; then
  curl -L -o /usr/local/bin/jq https://github.com/jqlang/jq/releases/download/jq-1.7.1/jq-linux-amd64
  chmod +x /usr/local/bin/jq
fi
jq --version
'''
      }
    }

    stage('Generate Signing Keys') {
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail
set -x
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
witness run \
  --step build \
  --signer-file-key-path testkey.pem \
  --enable-archivista \
  --archivista-server "${ARCHIVISTA_URL}" \
  -o attestations/build.json -- \
  bash -lc 'echo "Simulated build complete"'
'''
        stash name: 'witness-out', includes: 'dist/**,attestations/**,.witness_archi_subcmd,testpub.pem'
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

SUBJECT_FLAGS=""
while IFS=$'\\t' read -r n d; do
  [ -z "$n" ] || [ -z "$d" ] && continue
  SUBJECT_FLAGS+=" --subjects ${n}=${d}"
done < <(jq -r '.subject[] | [.name, .digest.sha256] | @tsv' "$DEC")

echo "Local subjects:"
jq -r '.subject[] | [.name, .digest.sha256] | @tsv' "$DEC" || true

witness verify "$ATT" --publickey testpub.pem ${SUBJECT_FLAGS}
'''
      }
    }

    stage('Verify Attestation (remote)') {
      steps {
        unstash 'witness-out'
        sh '''#!/usr/bin/env bash
set -euo pipefail
set -x
SUBCMD=$(cat .witness_archi_subcmd || echo none)
if [ "$SUBCMD" = "none" ]; then
  echo "Failing: expected archivist/archivista subcommand. Set a version that includes it."
  exit 1
fi

REMOTE=attestations/remote/build-remote.json
DEC_REMOTE=attestations/remote/build-remote.decoded.json
mkdir -p attestations/remote

witness "$SUBCMD" get \
  --archivista-server "${ARCHIVISTA_URL}" \
  --step build \
  --output "${REMOTE}"

jq -r '.payload' "$REMOTE" | base64 -d > "$DEC_REMOTE"

REMOTE_SUBJECT_FLAGS=""
while IFS=$'\\t' read -r n d; do
  [ -z "$n" ] || [ -z "$d" ] && continue
  REMOTE_SUBJECT_FLAGS+=" --subjects ${n}=${d}"
done < <(jq -r '.subject[] | [.name, .digest.sha256] | @tsv' "$DEC_REMOTE")

echo "Remote subjects:"
jq -r '.subject[] | [.name, .digest.sha256] | @tsv' "$DEC_REMOTE" || true

witness verify "$REMOTE" --publickey testpub.pem ${REMOTE_SUBJECT_FLAGS}
'''
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'attestations/**/*.json, dist/**, .witness_archi_subcmd, testpub.pem', fingerprint: true
    }
  }
}
