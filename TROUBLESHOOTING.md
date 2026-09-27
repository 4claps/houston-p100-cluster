# Troubleshooting

Known issues and workarounds for running llama.cpp on the Houston 3x Tesla P100 rig.

## `-sm tensor` crashes on second request (GGML_ASSERT(bcj.nodes[i]))

**TL;DR:** `--split-mode tensor` loads the model and successfully serves one request, then crashes on the *next* request with `GGML_ASSERT(bcj.nodes[i]) failed` inside `ggml-backend-meta.cpp`. Reproduces with 2 or 3 P100s, regardless of context size, MTP, or prompt-cache settings. `--split-mode layer` does not hit this and is the current stable config for this build.

Tracked upstream: [ggml-org/llama.cpp#29466](https://github.com/ggml-org/llama.cpp/issues/29466)

**Workaround:** run with `-sm layer` instead of `-sm tensor` until this is fixed upstream.
