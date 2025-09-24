# Viral Moments Service Docs

This repository publishes the shareable documentation for the Viral Moments microservice. All documents are generated from the main service codebase and collected under the `docs/` folder.

## Contents

- `docs/RunPodIntegration.md` – RunPod deployment & operations guide with webhook contracts and troubleshooting.
- `docs/ViralMomentResponse.md` – Source-of-truth schema for the `ServiceResult` payload and nested `ViralMoment` models.
- `docs/ViralMomentsMicroservice.md` – Architecture overview, module parity with the Laravel pipeline, and configuration table.
- `docs/README.md` – Bundle index with suggested usage notes.

## Updating

1. Pull the latest service repository.
2. Refresh the markdown files (edit in the service repo).
3. Rebuild the docs bundle and sync it here.
4. Commit and push changes to share with stakeholders.

> These docs are intended for platform integrators, product, and partner teams who need the contract without cloning the full service repo.
