# confidential-secret-demo — Part B workload tenant

A minimal "tenant" repo for the confidential secrets vault **sigstore-provenance**
path (Part B). Tagging this repo measures the deployment and publishes a
`snp-tdx-multiplatform` attestation **under this repo** — which is exactly what the
vault's `snpVerifier` checks: it proves the running enclave's measurement was built
from this repo, then binds that to the live SEV-SNP quote before releasing secrets.

```
tag this repo ─▶ tyurek/measure-image-action ─▶ snp-tdx-multiplatform attestation
                                                 (measurement, under this repo)
                                                          │
  vault put DEMO_SECRET --repo tyurek/confidential-secret-demo
                                                          ▼
  dev-launch on box2 ─▶ stage 3b sends the quote ─▶ vault: sigstore(this repo) ==
  quote measurement ─▶ seals DEMO_SECRET to the enclave ─▶ container gets it
```

## Files
- `tinfoil-config.yml` — the **measured** config (its sha256 is the `tinfoil-config-hash`).
- `external-config.yml` — host-authored, **not** measured; the `vault:` block for dev-launch.
- `.github/workflows/release.yml` — tag → `tyurek/measure-image-action@main` → attest + release.

## Prereqs
- `tyurek/cvmimage` has a release that **includes the split raw** (the `release.yml`
  zstd+split change). Set `cvm-version` in `tinfoil-config.yml` to that version.
- `tyurek/measure-image-action@main` is built (action.yaml → the fork's ghcr image).

## 1. Publish the attestation
```bash
git tag v0.0.1 && git push origin v0.0.1     # triggers release.yml → measure → attest → release
```
This creates a release with `tinfoil-deployment.json` and a sigstore attestation of it
under `tyurek/confidential-secret-demo`.

## 2. Dry-run on box2

> **Must be non-debug.** `measure-image-action` measures the *clean* cmdline. tinfoil's
> *debug* mode appends `tinfoil-debug=on …` and injects an SSH container (changing the
> config-hash), which would change the measurement and fail the `code == enclave` check.
> So launch with `debug:false` — and verify via the shim, not SSH.

```bash
V=0.0.0.6; R=tyurek/cvmimage; IMG=/mnt/large/tinfoil/images   # box2 ImageDir

# a) fetch the cvmimage-fork artifacts + reassemble the raw into the ImageDir
#    (so tinfoild's FetchLegacy cache-hits instead of hitting images.tinfoil.sh)
gh release download "v$V" -R "$R" -D /tmp/cvm -p '*'
sudo cp /tmp/cvm/tinfoil-inference-v$V.vmlinuz /tmp/cvm/tinfoil-inference-v$V.initrd "$IMG"/
cat /tmp/cvm/tinfoil-inference-v$V.raw.zst.part.* | zstd -d | sudo tee "$IMG/tinfoil-inference-v$V.raw" >/dev/null
# verify the reassembled raw matches the attested manifest
test "$(sha256sum $IMG/tinfoil-inference-v$V.raw | awk '{print $1}')" = \
     "$(jq -r .raw /tmp/cvm/tinfoil-inference-v$V-manifest.json)" && echo "raw OK"
ROOT=$(jq -r .root /tmp/cvm/tinfoil-inference-v$V-manifest.json)

# b) run the vault in sigstore mode (default verifier — no -pin-measurement/-dev-verify)
./svault -addr 0.0.0.0:8099 -identity vault_identity.json &   # needs egress to GitHub + KDS
KEY=$(grep -oE 'vault HPKE key: [0-9a-f]+' nohup.out | awk '{print $4}')

# c) store the secret under THIS repo
tinfoil-cli vault put DEMO_SECRET --value 's3cret' \
  --repo tyurek/confidential-secret-demo --vault http://localhost:8099 --vault-hpke-key "$KEY"

# d) dev-launch NON-debug. The cmdline must equal measure.py's exactly:
HASH=$(sha256sum tinfoil-config.yml | awk '{print $1}')
CMDLINE="readonly=on pci=realloc,nocrs modprobe.blacklist=nouveau nouveau.modeset=0 root=/dev/mapper/root roothash=${ROOT} tinfoil-config-hash=${HASH}"
jq -n --arg cmd "$CMDLINE" --arg cfg "$(base64 -w0 tinfoil-config.yml)" \
      --arg ext "$(cat external-config.yml)" \
  '{name:("secret-demo-"+ (now|floor|tostring)), cpus:4, memory:4096, debug:false,
    skip_manifest:true, config:$cfg, external_config:$ext, custom_cmdline:$cmd}' \
| curl -s -X POST http://localhost:8080/dev-launch -H 'Content-Type: application/json' --data-binary @-
```

## 3. Verify
- `journalctl`/vault log shows `released 1 secret(s) for tyurek/confidential-secret-demo`,
  and the deployment reaches `ready`.
- Through the shim (no SSH): `curl -k https://localhost:<http_port>/secret-check` → `DEMO_SECRET len=6`.
- The host only ever saw the secret **name** + a release count — never the value.

If the `code == enclave` measurement check fails, the cmdline didn't match: confirm
`debug:false`, the exact `roothash` (manifest `root`), and `tinfoil-config-hash` =
`sha256(tinfoil-config.yml)`.
