stage('Create & Sign Policy') {
  steps {
    sh '''#!/usr/bin/env bash
set -euo pipefail
set -x

PUBKEY_PEM_B64="$(base64 -w0 testpub.pem 2>/dev/null || openssl base64 -A < testpub.pem)"
KEYID="$(printf '%s' "${PUBKEY_PEM_B64}" | base64 -d | sha256sum | awk '{print $1}')"

ATT_FILE="attestations/build.json"
if [ ! -r "${ATT_FILE}" ]; then
  echo "attestations/build.json missing or not readable" >&2
  ls -l attestations || true
  exit 1
fi

PRED_TYPE="$(grep -o '"predicateType"[[:space:]]*:[[:space:]]*"[^"]*"' "${ATT_FILE}" 2>/dev/null | head -n1 | cut -d\" -f4 || true)"

if [ -z "${PRED_TYPE}" ]; then
  PAYLOAD_B64="$(grep -o '"payload"[[:space:]]*:[[:space:]]*"[A-Za-z0-9+/=]*"' "${ATT_FILE}" 2>/dev/null | head -n1 | cut -d\" -f4 || true)"
  if [ -n "${PAYLOAD_B64}" ]; then
    PRED_TYPE="$(printf '%s' "${PAYLOAD_B64}" | base64 -d 2>/dev/null | grep -o '"predicateType"[[:space:]]*:[[:space:]]*"[^"]*"' | head -n1 | cut -d\" -f4 || true)"
  fi
fi

if [ -z "${PRED_TYPE}" ]; then
  echo "Failed to extract predicateType from ${ATT_FILE}" >&2
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

sed -i "s|KEYID_PLACEHOLDER|${KEYID}|g" policy.json
sed -i "s|PUBKEY_BASE64_PLACEHOLDER|${PUBKEY_PEM_B64}|g" policy.json
sed -i "s|PRED_PLACEHOLDER|${PRED_TYPE}|g" policy.json

witness sign --signer-file-key-path testkey.pem -f policy.json -o policy-signed.json
'''
  }
}
