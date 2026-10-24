# deroute-core-config

Public [Tinfoil Containers](https://docs.tinfoil.sh/containers/overview) configuration and measured-release repository for **deroute-core**, the attested enclave backend of [deRoute](https://drouter-demo.vercel.app).

Every release in this repository is measured by Tinfoil's GitHub Actions workflow (`tinfoil-release.yml` -> `tinfoil-release-publish.yml`) and recorded in the public Sigstore Rekor transparency log so any browser or independent verifier can verify the exact container image digest, hardware measurement, network egress allowlist, and attested Ed25519 signing keys before sending a request.

## What runs in this enclave

The `tinfoil-config.yml` file in this repository declares a single stateless container (`deroute-core`) running on an attested Confidential VM (`cvm-version: 0.14.7`, 2 vCPUs, 8192 MB RAM) with a read-only root filesystem (`read_only: true`, `/tmp` tmpfs only):

1. **Whole-library knowledge retrieval (`GET /knowledge/corpus`)**: Returns the entire public confidential-computing knowledge base (`drouter.enclave_intelligence`) with 768-dimensional embeddings and a canonical SHA-256 digest (`contentSha256`). Accepts zero query parameters so MongoDB Atlas never learns what any user is looking for.
2. **Content-free Gemini control-plane planner (`POST /agent/plan`)**: Accepts only a bucketed `TaskDescriptor` (`taskKind`, `$INPUT` handle, power-of-two `inputTokenEstimate`, `redactionCounts` capped at 5 per category, boolean `verification` flags, and `allowedTools`). Enforces the `assertContentFree` egress firewall on every outbound request to `generativelanguage.googleapis.com` (`store: false`) and signs a canonical egress receipt (`{ sha256, epochHourSec, kind: "gemini-egress" }`) with the enclave-generated `egress-signing` Ed25519 key.
3. **Transparency log tree-head signer (`POST /transparency/sign-tree-head`, internal / operator-authenticated)**: Not exposed on the public `shim.paths`. Requires a constant-time Bearer token matching `CORE_OPERATOR_TOKEN` (fails closed when unset) and recomputes the RFC 6962 Merkle root directly from `drouter.transparency_log` in MongoDB to verify `(treeSize, rootHash)` before signing with `tree-signing`.
4. **Attested public key discovery (`GET /keys` and `GET /health`)**: Exposes the raw 32-byte hex and 44-byte RFC 8410 SPKI DER hex public keys for `egress-signing` and `tree-signing`, plus `publicKeysSha256` (`sha256(canonicalJson({ egress, tree }))`), so clients can cross-check them against Tinfoil `crypto_material` (`https://tinfoil.sh/key/spki/v1`) or a platform attestation report (Google Confidential Space token nonce or Intel TDX `report_data`).

## Attested keys and explicit `CORE_KEY_SOURCE`

`deroute-core` requires an explicit, platform-neutral `CORE_KEY_SOURCE` environment variable and refuses to start if it is missing or unknown:

- `CORE_KEY_SOURCE=tinfoil` (configured in `tinfoil-config.yml`): Loads the two Ed25519 key pairs generated at enclave boot by the Tinfoil CVM and mounted read-only at `/run/tinfoil/keys/<id>/`:
  - `egress-signing` (`ed25519`): Signs the canonical SHA-256 receipt of every outbound Gemini control-plane request.
  - `tree-signing` (`ed25519`): Signs verified deRoute transparency log Merkle tree heads.
- `CORE_KEY_SOURCE=boot`: Generates both Ed25519 key pairs in memory at boot (never written to disk) and exposes `publicKeysSha256` (`sha256(canonicalJson({ egress, tree }))`) on `GET /keys` so a platform attestation quote can bind them.

Neither private key ever leaves the enclave.

## Egress allowlist (`core-egress`)

Outbound traffic from the enclave is blocked by default and restricted by the `core-egress` allowlist network to the 5 hosts the service actually contacts:

- **MongoDB Atlas (`drouter.enclave_intelligence` and `drouter.transparency_log`)**:
  - `drouter.0mvrm0i.mongodb.net`
  - `ac-cdskmit-shard-00-00.0mvrm0i.mongodb.net`
  - `ac-cdskmit-shard-00-01.0mvrm0i.mongodb.net`
  - `ac-cdskmit-shard-00-02.0mvrm0i.mongodb.net`
- **Google Gemini Interactions API**:
  - `generativelanguage.googleapis.com`

## Secrets

- `MONGODB_URI`: MongoDB Atlas connection string for reading `drouter.enclave_intelligence` and verifying `drouter.transparency_log`.
- `GEMINI_API_KEY`: Google Gemini API key used exclusively for content-free control-plane planning (`POST /agent/plan`).
- `CORE_OPERATOR_TOKEN`: Constant-time Bearer token required by `POST /transparency/sign-tree-head`.

## How browsers verify this enclave

Browsers connect using `SecureClient` from the [`tinfoil`](https://www.npmjs.com/package/tinfoil) SDK:

```ts
import { SecureClient } from "tinfoil";

const client = new SecureClient({
  enclaveURL: "https://<instance>.<org>.containers.tinfoil.dev",
  configRepo: "thepigdestroyer/deroute-core-config",
});

await client.ready();
const verificationDoc = await client.getVerificationDocument();
```

Before sending any request, `SecureClient`:

1. Fetches the hardware attestation quote from the enclave and verifies the AMD SEV-SNP / Intel TDX signature chain back to the silicon vendor root.
2. Fetches the latest signed release from `thepigdestroyer/deroute-core-config` and verifies its Sigstore Rekor transparency log inclusion proof.
3. Confirms the running enclave measurement matches the measured `tinfoil-config.yml` and pinned `ghcr.io/thepigdestroyer/deroute-core@sha256:...` image digest.
4. Binds TLS / HPKE keys and `crypto_material` (`egress-signing` and `tree-signing` SPKI public keys) to the verified enclave report.

## Releasing a new version

1. Build and push `ghcr.io/thepigdestroyer/deroute-core` via `.github/workflows/core-image.yml` in the `drouter-demo` repository.
2. Copy the resulting `sha256:<digest>` into `tinfoil-config.yml` in this repository and commit to `main`.
3. Trigger the **Tinfoil Release** workflow:
   ```bash
   gh workflow run tinfoil-release.yml -f version=v0.0.1
   ```
4. Once `tinfoil-release-publish.yml` finishes publishing the signed measurement to GitHub Releases and Sigstore Rekor, deploy or update the container in the Tinfoil dashboard.
