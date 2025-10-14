pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
  }

  parameters {
    string(name: 'ARCHIVISTA_URL', defaultValue: 'http://archivista:8082', description: 'Archivista server URL')
    string(name: 'WITNESS_VERSION', defaultValue: 'v0.5.5', description: 'Witness CLI version tag')
    booleanParam(name: 'VERIFY_ATTESTATION', defaultValue: true, description: 'Run verification stage')
  }

  environment {
    WITNESS_BIN = "${WORKSPACE}/.witness/witness"
    PATH = "${WORKSPACE}/.witness:${PATH}"
  }

  stages {

    stage('Prep') {
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail
mkdir -p .witness dist attestations
'''
      }
    }

    stage('Install Witness CLI (cached)') {
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail
if [[ ! -x .witness/witness ]]; then
  echo "Downloading Witness ${WITNESS_VERSION}"
  curl -sSL https://raw.githubusercontent.com/in-toto/witness/main/install-witness.sh -o install-witness.sh
  chmod +x install-witness.sh
  ./install-witness.sh "${WITNESS_VERSION}"
  mv witness .witness/
fi
.witness/witness version
'''
      }
    }

    stage('Generate Demo Keys') {
      when { not { credentials('witness-ed25519-key') } }
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail
openssl genpkey -algorithm ed25519 -out testkey.pem
openssl pkey -in testkey.pem -pubout > testpub.pem
'''
      }
    }

    stage('Prepare Keys (prod)') {
      when { credentials('witness-ed25519-key') }
      steps {
        withCredentials([file(credentialsId: 'witness-ed25519-key', variable: 'WIT_KEY')]) {
          sh '''#!/usr/bin/env bash
set -euo pipefail
cp "$WIT_KEY" testkey.pem
openssl pkey -in testkey.pem -pubout > testpub.pem
'''
        }
      }
    }

    stage('Preflight Archivista') {
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail
curl -sSf "${ARCHIVISTA_URL}/health" || { echo "Archivista not reachable"; exit 1; }
'''
      }
    }

    stage('Build & Attest') {
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail
echo "hello $(date -u +%FT%TZ)" > dist/app.txt
witness run \
  --step build \
  --signer-file-key-path testkey.pem \
  --enable-archivista \
  --archivista-server "${ARCHIVISTA_URL}" \
  -o attestations/build.json -- \
  bash -lc 'echo "Simulated build complete"'
jq 'del(.predicate.materials? // empty)' attestations/build.json > attestations/build.slim.json
'''
        stash name: 'witness-out', includes: 'dist/**,attestations/**'
      }
    }

    stage('Verify Attestation') {
      when { expression { return params.VERIFY_ATTESTATION } }
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail
witness verify \
  --archivista-server "${ARCHIVISTA_URL}" \
  --public-key testpub.pem \
  --predicate-type slsaprovenance/v1 \
  attestations/build.json
'''
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'attestations/**/*.json, dist/**', fingerprint: true
      sh 'sha256sum attestations/build.json || shasum -a256 attestations/build.json'
    }
    success {
      echo 'Pipeline succeeded.'
    }
    failure {
      echo 'Pipeline failed.'
    }
  }
}
