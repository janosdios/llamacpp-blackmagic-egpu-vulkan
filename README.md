# llama.cpp on a Blackmagic eGPU (Intel Mac, macOS Ventura)
### Sharing the summary of a Claude-code session to make an old Blackmagic eGPU capable to run some smaller AI model on macOS Ventura.
---

How to turn an Intel MacBook Pro + Blackmagic eGPU into a working local LLM / vision node, without leaving macOS.

**Setup used:** MacBook Pro 2017 (Intel, dual-core), macOS Ventura 13, Blackmagic eGPU (Radeon Pro 580, 8 GB), models on an external SSD. Built November 2026.

## Key finding: use Vulkan, not Metal

llama.cpp's default Metal backend runs correctly on the AMD card but is slower than the CPU. The Vulkan backend via MoltenVK is several times faster.

| Backend (Qwen3-4B Q4_K_M) | Prompt | Generation |
|---|---:|---:|
| Metal (eGPU) | 15 t/s | 3.7 t/s |
| CPU only | 18 t/s | 6.3 t/s |
| **Vulkan via MoltenVK (eGPU)** | **227 t/s** | **25 t/s** |

## 1. Check the prerequisites

```
xcode-select -p; git --version; cmake --version; brew --version
```

You need the Command Line Tools, git, cmake and Homebrew. The full Xcode app is **not** needed.

## 2. Install the Vulkan build dependencies

```
brew install vulkan-loader vulkan-headers shaderc spirv-headers
```

Don't install `molten-vk` or `vulkan-tools` from Homebrew. On Ventura/Intel (a Homebrew Tier 3 configuration) both require the full Xcode app to build. `shaderc` compiles from source and takes about 8 minutes.

## 3. Install MoltenVK from the official prebuilt release

1. Download `MoltenVK-macos.tar` from https://github.com/KhronosGroup/MoltenVK/releases (Assets of the latest release).
2. Extract it, remove the Gatekeeper quarantine flag, and keep only the two files that matter:

```
cd ~ && tar -xf ~/Downloads/MoltenVK-macos.tar
xattr -dr com.apple.quarantine ~/MoltenVK
mkdir -p ~/.local/lib/moltenvk
cp ~/MoltenVK/MoltenVK/dynamic/dylib/macOS/{libMoltenVK.dylib,MoltenVK_icd.json} ~/.local/lib/moltenvk/
mv ~/MoltenVK ~/.Trash/ && mv ~/Downloads/MoltenVK-macos.tar ~/.Trash/
```

Without the `xattr` step, macOS blocks the library as "malicious software", and "Allow anyway" in Privacy settings doesn't help. The two files must stay in the same folder, because the `.json` refers to the library by a relative path.

## 4. Make the Vulkan settings permanent

```
echo 'export VK_ICD_FILENAMES="$HOME/.local/lib/moltenvk/MoltenVK_icd.json"' >> ~/.zshrc
echo 'export MVK_CONFIG_LOG_LEVEL=1' >> ~/.zshrc
source ~/.zshrc
```

`MVK_CONFIG_LOG_LEVEL=1` hides MoltenVK's long extension list and shows only errors.

## 5. Build llama.cpp with Vulkan

```
conda deactivate   # only if you use conda; keeps its libraries out of the build
cd ~ && git clone https://github.com/ggml-org/llama.cpp && cd llama.cpp
cmake -B build-vk -DGGML_METAL=OFF -DGGML_VULKAN=ON -DLLAMA_BUILD_TESTS=OFF
cmake --build build-vk --config Release -j 4
```

`-DLLAMA_BUILD_TESTS=OFF` is required. One test file fails to compile against the macOS 13.3 SDK, and the tests aren't needed anyway. The build takes 20+ minutes on a dual-core machine, mostly for compiling the Vulkan shaders.

## 6. Verify that Vulkan sees the eGPU

```
./build-vk/bin/llama-cli --list-devices
```

Expected:

```
Vulkan0: AMD Radeon Pro 580 (8192 MiB, 8192 MiB free)
Vulkan1: Intel(R) Iris(TM) Plus Graphics 650 ...
```

Always pass `-dev Vulkan0` so nothing gets spread onto the Intel iGPU.

## 7. Download a model

Qwen3.5-4B with its vision projector worked best on this card:

```
D=/Volumes/.../llamacpp   # adjust to your models folder
hf download unsloth/Qwen3.5-4B-GGUF --include "*Q8_0*" --local-dir $D
hf download unsloth/Qwen3.5-4B-GGUF --include "mmproj-F16*" --local-dir $D
mv $D/mmproj-F16.gguf $D/Qwen3.5-4B-mmproj-F16.gguf
```

Use the **F16** projector, not BF16, because this card has no bf16 support. Rename the projector right away: every multimodal model ships an `mmproj-F16.gguf`, so the generic names collide.

## 8. Benchmark

```
./build-vk/bin/llama-bench -m $D/Qwen3.5-4B-Q8_0.gguf -ngl 99 -dev Vulkan0 -t 1 --poll 0
```

| Qwen3.5-4B | Size | Prompt (pp512) | Generation (tg128) |
|---|---:|---:|---:|
| Q4_K_M | 2.5 GiB | 201 t/s | 21.4 t/s |
| **Q8_0** | 4.2 GiB | **213 t/s** | **17.0 t/s** |

Q8_0 costs only about 20% generation speed for near-lossless quality, so it's the better default.

## 9. Store the API key in the Keychain

```
security add-generic-password -a "$USER" -s llama-api-key -w "$(openssl rand -hex 24)"
security find-generic-password -a "$USER" -s llama-api-key -w   # shows the key, for your clients
```

The key is encrypted at rest. The server script reads it at startup, and it never appears in a file or in the process list.

## 10. The server script

`~/llama-scripts/qwen35-4b-vision.sh`:

```
#!/bin/zsh
export VK_ICD_FILENAMES="$HOME/.local/lib/moltenvk/MoltenVK_icd.json"
export MVK_CONFIG_LOG_LEVEL=1

MODELS=/Volumes/.../llamacpp
[ -d "$MODELS" ] || { echo "SSD not mounted: $MODELS"; exit 1; }
system_profiler SPDisplaysDataType | grep -q "Radeon Pro 580" || { echo "eGPU not connected"; exit 1; }

LLAMA_API_KEY="$(security find-generic-password -a "$USER" -s llama-api-key -w)" \
  || { echo "API key not found in Keychain"; exit 1; }
export LLAMA_API_KEY

exec "$HOME/llama.cpp/build-vk/bin/llama-server" \
  -m "$MODELS/Qwen3.5-4B-Q8_0.gguf" \
  --mmproj "$MODELS/Qwen3.5-4B-mmproj-F16.gguf" \
  --alias qwen3.5-4b --jinja \
  --image-max-tokens 1024 \
  --chat-template-kwargs '{"enable_thinking":false}' \
  -ngl 99 -dev Vulkan0 -t 1 --poll 0 -np 1 \
  --temp 0.5 --top-p 0.8 --top-k 20 \
  -c 32768 \
  --host 0.0.0.0 --port 8080
```

```
chmod +x ~/llama-scripts/qwen35-4b-vision.sh
~/llama-scripts/qwen35-4b-vision.sh
```

What each non-obvious flag does:

| Flag | Why |
|---|---|
| `-t 1 --poll 0` | By default the CPU threads spin while waiting for the GPU. With these flags, the CPU peaked at 75 °C instead of 100 °C, at the same speed. |
| `-dev Vulkan0` | Keeps all work on the eGPU. |
| `--alias qwen3.5-4b` | Clients use a clean model name instead of the full file path. |
| `--jinja` | Required for the chat-template options; Gemma 4 crashes without it. |
| `--image-max-tokens 1024` | Image size cap. See the vision notes below. |
| `enable_thinking:false` | No thinking block by default. Clients can turn it on per request. |
| `-np 1` | One slot: the whole 32K context goes to a single request, and others queue. |
| `--temp 0.5 --top-p 0.8 --top-k 20` | Calmer sampling, so the model invents fewer details in image descriptions. |
| eGPU check | Without the eGPU, the Intel iGPU would become `Vulkan0`. |

## 11. Test from another machine

```
curl http://<macbook-name>:8080/v1/models                                   # expect 401
curl http://<macbook-name>:8080/v1/models -H "Authorization: Bearer <key>"  # expect the model list
```

The built-in web UI is at `http://<macbook-name>:8080`. Enter the key in its settings.

## Vision notes

MoltenVK doesn't support flash attention, so llama.cpp falls back to regular attention. The startup warnings about this, and about "CLIP graph uses unsupported operators", are expected. The vision encoder works, but it's the slow part. Its cost grows with the square of the image size:

| `--image-max-tokens` | Encoder input | Encoding time | Accuracy |
|---:|---|---:|---|
| 512 | ~720×720 px | ~25 s | Good, occasional invented details |
| 1024 | ~1024×1024 px | ~77 s | Accurate |

Gemma 4 E2B was tested as an alternative. It was slower (~99 s encoding at its default size), less accurate, and thinks by default.

## Results summary

| | |
|---|---|
| Model | Qwen3.5-4B Q8_0 with vision |
| Text | ~17 t/s generation, ~210 t/s prompt processing |
| Images | ~77 s per image at full accuracy |
| CPU temperature | ~75 °C under load |
| Access | API key from the Keychain, any machine on the network |
