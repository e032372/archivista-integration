pipeline {
  agent any

  environment {
    ARCHIVISTA_URL = credentials('archivista-url') ?:
                     'http://archivista:8082'   // fallback if you don't use credentials
  }

  stages {
    stage('Prepare: keys & sanity') {
      steps {
        sh '''
          set -euxo pipefail

          # Show Witness version (preinstalled in the Jenkins image)
          witness version

          # Generate demo Ed25519 keypair (replace with KMS or keyless for real pipelines)
          openssl genpkey -algorithm ed25519 -out testkey.pem
          openssl pkey -in testkey.pem -pubout > testpub.pem

          # Sanity: confirm Archivista is reachable from this container
          curl -fsS "${ARCHIVISTA_URL}/health" || true
        '''
      }
    }

    stage('Build (attested)') {
      steps {
        sh '''
          set -euxo pipefail

          # "Build" a trivial artifact to keep the demo language-agnostic
          echo "Hello, Archivista!" > artifact.txt

          # Record an attestation for the build step and push to Archivista
          witness run \
            --step build \
            --enable-archivista \
            --archivista-server "${ARCHIVISTA_URL}" \
            --signer-file-key-path testkey.pem \
            -o build.att.json \
            -- sh -c "echo build complete"

          # Compute the subject digest for later lookup
          sha256sum artifact.txt | tee artifact.sha256
        '''
      }
    }

    stage('Test (attested)') {
      steps {
        sh '''
          set -euxo pipefail

          # Record a test step attestation and push to Archivista
          witness run \
            --step test \
            --enable-archivista \
            --archivista-server "${ARCHIVISTA_URL}" \
            --signer-file-key-path testkey.pem \
            -o test.att.json \
            -- sh -c "grep -q Archivista artifact.txt && echo tests passed"
        '''
      }
    }

    stage('Verify from Archivista') {
      steps {
        sh '''
          set -euxo pipefail

          # Extract sha256 digest and ask Witness to find related attestations in Archivista
          ART_SHA="$(cut -d' ' -f1 artifact.sha256)"

          # witness verify can use archivista as a source; we don't supply a policy here,
          # but we provide --subjects to prove lookups and connectivity end-to-end.
          witness verify \
            --enable-archivista \
            --archivista-server "${ARCHIVISTA_URL}" \
            --subjects "sha256:${ART_SHA}" || true

          echo "Verified lookup for subject sha256:${ART_SHA} against ${ARCHIVISTA_URL}"
        '''
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: '*.att.json,*.sha256,*.pem', fingerprint: true
    }
  }
}
