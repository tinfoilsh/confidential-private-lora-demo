# Confidential Private LoRA Serving Demo

A [Tinfoil Container](https://docs.tinfoil.sh/containers/overview) serving
[openai/gpt-oss-20b](https://huggingface.co/openai/gpt-oss-20b) together with a
**private** LoRA fine-tune on one attested endpoint, using a stock upstream
[vLLM](https://github.com/vllm-project/vllm) image pinned by digest.

This is the private-weights variant of
[confidential-lora-demo](https://github.com/tinfoilsh/confidential-lora-demo):
same base model, same serving stack, but the adapter weights never appear in
plaintext outside the enclave.

## How the adapter stays private

The adapter is packaged with
[modelwrap](https://github.com/tinfoilsh/modelwrap) as an **encrypted model
pack** (`emwp`): an EROFS image protected by dm-crypt underneath dm-verity.
The infrastructure only ever holds ciphertext.

At boot, the enclave:

1. builds fresh hardware attestation evidence (Intel TDX + NVIDIA GPU),
2. presents it to the model owner's keyserver (`kbs-url` above, a
   [tinfoilsh/keyserver](https://github.com/tinfoilsh/keyserver) instance),
3. receives `LORA_MODEL_KEY` only if the evidence matches the release
   measurements the owner pinned for this exact repo and tag, and
4. decrypts and dm-verity-mounts the adapter inside the enclave.

The key is used by the boot process only — it is never exposed to the
workload container. Swapping the base model, the adapter ciphertext, the
serving image, or the keyserver URL all change the measurement, so the
keyserver refuses to release the key to anything but this exact release.

## Call it

The endpoint exposes an OpenAI-compatible API. The base model and the
private fine-tune are separate model names on the same deployment:

```bash
# List models — shows the base and the adapter
curl https://<deployment-domain>/v1/models

# Base model
curl https://<deployment-domain>/v1/chat/completions \
  -H "Authorization: Bearer $TINFOIL_API_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"model": "gpt-oss-20b", "messages": [{"role": "user", "content": "..."}]}'

# Private LoRA fine-tune
curl https://<deployment-domain>/v1/chat/completions \
  -H "Authorization: Bearer $TINFOIL_API_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"model": "gpt-oss-20b-text2sql", "messages": [{"role": "user", "content": "..."}]}'
```

## Verify it

```bash
tinfoil attestation verify -e <deployment-domain> -r tinfoilsh/confidential-private-lora-demo
```

This checks the hardware attestation against the measurements published for
this repository's release, which bind the exact `tinfoil-config.yml` above —
image digest, base model hash, adapter ciphertext hash, and keyserver URL
included.

## Deploy your own

1. Wrap your adapter: `modelwrap --model-dir <adapter> --encrypt --key-file
   <64-byte-key> --verify --output <dir> <your/model-identity>` and place the
   resulting `.emwp` on the target host. Only ciphertext leaves your
   infrastructure.
2. Run a [keyserver](https://github.com/tinfoilsh/keyserver) holding the key,
   with a policy pinning this repo, release tag, and deployment domain.
3. Point `models:` and `kbs-url:` in `tinfoil-config.yml` at your artifact
   and keyserver, push a tag, run the release workflow, and deploy from the
   [dashboard](https://dash.tinfoil.sh) or the `tinfoil` CLI.
