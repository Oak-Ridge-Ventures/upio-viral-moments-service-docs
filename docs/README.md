# Viral Moments Service Documentation Bundle

Curated documentation set for sharing outside the core repository. These files describe how to run the Viral Moments microservice on RunPod, the full response contract, and the architecture it mirrors from Laravel.

## Included Guides

1. **RunPodIntegration.md** – End-to-end instructions for deploying and operating the service on RunPod Serverless, including webhook payloads, configuration overrides, and troubleshooting.
2. **ViralMomentResponse.md** – Field-by-field specification of the `ServiceResult` and `ViralMoment` models, including pricing telemetry and sample payloads.
3. **ViralMomentsMicroservice.md** – Architectural overview, module-level parity with the Laravel pipeline, and operational considerations.

## Suggested Usage

- Publish this folder to the dedicated documentation repository (`upio-viral-moments-service-docs`).
- Share `RunPodIntegration.md` with platform engineers standing up RunPod endpoints.
- Use `ViralMomentResponse.md` as the contract reference for Laravel ingestion and QA validation.
- Keep `ViralMomentsMicroservice.md` handy for onboarding developers or reviewing feature parity.

> These documents are generated from the main service repository; update them there first before refreshing the bundle.
