// Jenkinsfile
pipeline {
  agent any

  environment {
    ARCHIVISTA_URL = 'http://archivista:8082'
    // Optionally pin; leave blank to auto-latest. Example: WITNESS_VERSION = 'v0.4.3'
    // WITNESS_VERSION = 'v0.4.3'
  }

  stages {

    stage('Install Tools') {
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail
set -x

# ---- Install Witness (with tag validation) ----
REQ_TAG="${WITNESS_VERSION:-}"
if [ -n "$REQ_TAG" ]; then
  if git ls-remote --exit-code --tags https://github.com/in-toto/witness "refs/tags/${REQ_TAG}" >/dev/null 2>&1; then
    USE_TAG="$REQ_TAG"
  else
    echo "Requested tag ${REQ_TAG} not found. Discovering latest..."
    USE_TAG=$(git ls-remote --tags https://github.com/in-toto/witness \
      | awk -F/ '/refs\\/tags\\/v[0-9]/{print $3}' \
      | grep -v '{}' | sort -V | tail -1)
    echo "Falling back to ${USE_TAG}"
  fi
else
  echo "No WITNESS_VERSION provided. Using latest tag."
  USE_TAG=$(git ls-remote --tags https://github.com/in-toto/witness \
    | awk -F/ '/refs\\/tags\\/v[0-9]/{print $3}' \
    | grep -v '{}' | sort -V | tail -1)
fi
echo "Installing Witness tag: ${USE_TAG}"

# Minimal Go toolchain if needed
if ! command -v go >/dev/null 2>&1; then
  curl -sSL https://go.dev/dl/go1.22.5.linux-amd64.tar.gz -o go.tgz
  tar -C /usr/local -xzf go.tgz
  export PATH=/usr/local/go/bin:$PATH
fi

# Try subdirectory form first; if it fails, try root
set +e
go install "github.com/in-toto/witness/cmd/witness@${USE_TAG}"
rc=$?
if [ $rc -ne 0 ]; then
  echo "Subdir install failed (rc=$rc). Trying root path..."
  go install "github.com/in-toto/witness@${USE_TAG}"
  rc=$?
fi
set -e
if [ $rc -ne 0 ]; then
  echo "Failed to install Witness at tag ${USE_TAG}"
  exit 1
fi

witness version || true

# ---- Detect archivist / archivista subcommand ----
HELP=$(witness help 2>&1 || true)
if echo "$HELP" | grep -E -q '\\barchivist\\b'; then
  echo archivist > .witness_archi_subcmd
elif echo "$HELP" | grep -E -q '\\barchivista\\b'; then
  echo archivista > .witness_archi_subcmd
else
  echo none > .witness_archi_subcmd
  echo "WARNING: No archivist/archivista subcommand present in ${USE_TAG}"
fi

# ---- jq ----
if ! command -v jq >/dev/null 2>&1; then
  curl -L -o /usr/local/bin/jq https://github.com/jqlang/jq/releases/download/jq-1.7.1/jq-linux-amd64
  chmod +x /usr/local/bin/jq
fi
jq --version
'''
      }
    }

    // (retain your remaining stages unchanged)
  }
}
