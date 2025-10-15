# Jenkins Pipeline for Archivista Integration

This repository contains a `Jenkinsfile` that defines a CI pipeline for building, attesting, and verifying artifacts using [Witness](https://github.com/in-toto/witness) with [Archivista](https://github.com/in-toto/archivista) integration.

## Pipeline Overview

The pipeline performs the following steps:

1. **Install Tools**  
   Installs Witness and `jq`. Detects if the Witness CLI supports the `archivist` or `archivista` subcommand.

2. **Generate Signing Keys (demo)**  
   Generates a demo Ed25519 key pair for signing attestations.

3. **Build & Attest (store in Archivista)**  
   Simulates a build, creates an attestation using Witness, and stores it in Archivista.

4. **Verify Attestation (from Archivista)**  
   Retrieves the attestation from Archivista and verifies it using the generated public key.

## Environment Variables

- `ARCHIVISTA_URL`: URL of the Archivista server (default: `http://archivista:8082`)
- `WITNESS_VERSION` (optional): Pin a specific Witness version.

## Artifacts

The pipeline archives:
- All generated attestations (`attestations/**/*.json`)
- Build outputs (`dist/**`)
- Policy files (`policy*.json`)
- Subcommand detection file (`.witness_archi_subcmd`)

## Requirements

- Jenkins with a Linux agent
- Internet access to download Witness and `jq`
- Access to an Archivista server

## Usage

This pipeline is ready to use in Jenkins. Adjust environment variables as needed for your setup.
