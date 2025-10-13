pipeline {
  agent any

  environment {
    // If Archivista is a container on the same Docker network (Option A):
    ARCHIVISTA_URL = 'http://archivista:8082'
    // If Archivista is a host process (Option B), use the line below instead:
    // ARCHIVISTA_URL = 'http://host.docker.internal:8082'
  }

  stages {
    stage('Install Witness CLI') {
      steps {
        sh '''
          bash -lc "
          set -euxo pipefail
          # Install Witness (from official docs)
          bash <(curl -s https://raw.githubusercontent.com/in-toto/witness/main/install-witness.sh)
          witness version
          "
        '''
      }
    }

    stage('Generate Signing Keys (demo)') {
      steps {
        sh '''
          bash -lc "
          set -euxo pipefail
          # Demo-only Ed25519 keypair for signing attestations.
          # In real pipelines, use KMS/Sigstore/OIDC keyless as appropriate.
          openssl genpkey -algorithm ed25519 -out testkey.pem
          openssl pkey -in testkey.pem -pubout > testpub.pem
          "
        '''
      }
    }

    stage('Build & Attest') {
      steps {
        sh '''
          bash -lc "
          set -euxo pipefail

          # Example "build" – replace with your real build command
          mkdir -p dist
          echo "hello $(date)" > dist/app.txt

          # Create an attestation and upload to Archivista
          # --enable-archivista + --archivista-server is the documented way
          witness run \
            --step build \
            --signer-file-key-path testkey.pem \
            --enable-archivista \
            --archivista-server "${ARCHIVISTA_URL}" \
            -o attestations/build.json -- \
            bash -lc 'echo "Simulated build complete"'
          "
        '''
      }
    }

    stage('Verify (optional policy)') {
      when { expression { return fileExists('testpub.pem') } }
      steps {
      sh '''#!/usr/bin/env bash
      set -euo pipefail
      set -x

      # Compute sha256 digest of the artifact and pass it as a subject
      DIGEST="$(sha256sum dist/app.txt | awk '{print $1}')"
      witness verify \
        --attestations attestations/build.json \
        -f dist/app.txt \
        -k testpub.pem
      '''
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'attestations/**/*.json, dist/**', fingerprint: true
    }
  }
}
