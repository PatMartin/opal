# OPAL contracts — release label v1.3.0

This archive contains the latest locally available five service contracts plus the separately drafted shared event schema.
Each filename contains `v1.3.0` and each OpenAPI `info.version` is `1.3.0`.

IMPORTANT: This is a version-label/repackaging release, **not** a completed feature reconciliation.
The five service contracts come from the latest single-setting-review package. The shared event contract is a separate draft: the full event emission v2 wire-format migration has NOT been applied to all five service contracts.
The `opal_think` source previously had an incomplete-source/reconstructed contract warning and must not be assumed authoritative.
The shared event envelope's own `metadata.schema_version: 2.0` is a separate WIRE version from OpenAPI `info.version: 1.3.0`.

Validation: six YAML documents parse, each declares OpenAPI 3.1.0 and info.version 1.3.0. Full semantic OpenAPI compliance, reference resolution and integration testing have not been performed.

## Files

| File | Previous info.version | Paths | SHA256 |
|---|---|---:|---|
| `opal_listen_v1.3.0.yaml` | `1.2.0` | 49 | `f486b6b60ab1e6e53443f1003239366bd53f577f30056cf9359f1c3d37aede1a` |
| `opal_talk_v1.3.0.yaml` | `1.2.0` | 27 | `d351a8a4bafb20cbb0cc9c0fb94f1575d42b22e2c98350a22c2bf0098757b169` |
| `opal_look_v1.3.0.yaml` | `1.2.0` | 45 | `aa94ad29690d02242a9c488af55fb67cab56aa0a18f72a66916c90c42b013a76` |
| `opal_think_v1.3.0.yaml` | `1.2.0` | 16 | `7d2389cdf82c1947158a0d91e2a9bfe0b79a4e734b1951ec26980c8f99f1bf12` |
| `opal_event_v1.3.0.yaml` | `1.2.0` | 20 | `f9047c4b8fce001562cfcc0ca46b56cbf6ca0467ec07d63189bfc2122548e17b` |
| `opal_events_shared_v1.3.0.openapi.yaml` | `2.0.0-draft` | 1 | `f9fee0ebf7fcce48077a5fbb3175ce1f90f3eec3c95e16092d2cde57065b0f0e` |
