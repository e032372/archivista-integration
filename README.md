# Jenkins Pipeline for Witness Attestation with Archivista

This project demonstrates a Jenkins pipeline that builds a simple artifact, generates cryptographic attestations using [Witness](https://github.com/in-toto/witness), and stores/verifies them with [Archivista](https://github.com/in-toto/archivista).

## Pipeline Overview

The pipeline performs the following steps:

1. **Install Tools**  
   Installs Witness and `jq` (if not already present).

2. **Generate Signing Keys (demo)**  
   Generates an Ed25519 key pair for signing attestations.

3. **Build & Attest (store in Archivista)**  
   - Builds a demo artifact (`dist/app.txt`).
   - Uses Witness to generate an attestation for the build.
   - Stores the attestation in Archivista.

4. **Verify Attestation (from Archivista)**  
   - Retrieves the attestation from Archivista.
   - Verifies the attestation using a minimal policy and the generated public key.

5. **Post Actions**  
   Archives all generated artifacts and attestations.

## Requirements

- Jenkins with a Linux agent
- Internet access to download tools
- Access to an Archivista server (default: `http://archivista:8082`)

## Environment Variables

- `ARCHIVISTA_URL`: URL of the Archivista server (default: `http://archivista:8082`)
- `WITNESS_VERSION`: (Optional) Pin Witness to a specific version

## Usage

1. Place the provided `Jenkinsfile` in your repository.
2. Configure your Jenkins job to use this pipeline.
3. Ensure the Archivista server is running and accessible.

## Notes

- The pipeline uses demo keys for signing. For production, use secure key management.
- The build step is a simple echo command for demonstration.
- All artifacts and attestations are archived for traceability.

---

For more information, see the [Witness documentation](https://github.com/in-toto/witness) and [Archivista documentation](https://github.com/in-toto/archivista).
