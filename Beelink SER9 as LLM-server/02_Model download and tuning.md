# Model Setup, MTP, GTT Tuning and Testing

Continuation guide for the dedicated AMD 890M llama.cpp server

Starting point: llama.cpp is already built under /opt/llama.cpp with Vulkan support, the dedicated llama service user exists and can access the AMD GPU, and /opt/llama-models exists.

## 1. Download the MTP-enabled test model

```
cd /opt/llama-models
```

```
sudo -u llama wget -O Qwen3.6-35B-A3B-UD-IQ4_NL.gguf \
"https://huggingface.co/unsloth/Qwen3.6-35B-A3B-MTP-GGUF/resolve/main/Qwen3.6-35B-A3B-UD-IQ4_NL.gguf"
```

```
ls -lh /opt/llama-models/Qwen3.6-35B-A3B-UD-IQ4_NL.gguf
```

## 2. Create the llama-server preset

Edit the preset file:

```
sudo nano /opt/llama.cpp/llama-server.conf
```

Use this known-good test configuration:

```
version = 1

[test]
model = /opt/llama-models/Qwen3.6-35B-A3B-UD-IQ4_NL.gguf
ctx-size = 131072
parallel = 1

cache-type-k = q8_0
cache-type-v = q8_0

spec-type = draft-mtp
spec-draft-n-max = 3
cache-type-k-draft = q8_0
cache-type-v-draft = q8_0

flash-attn = on
load-mode = mmap
jinja = true
load-on-startup = true
```

Notes: 128K context is 131072 tokens. Parallel is kept at 1 for MTP. Q8_0 is used for both the normal and MTP draft KV caches. GPU layer placement is left on automatic. Thread overrides are intentionally omitted because explicit 12/24 thread settings reduced generation speed substantially on this machine. mmap is retained because it was stable and not measurably slower than load-mode=none on this system.

## 3. Start the server manually

```
sudo -u llama /opt/llama.cpp/build/bin/llama-server \
  --models-preset /opt/llama.cpp/llama-server.conf \
  --models-max 1 \
  --host 0.0.0.0 \
  --port 8080
```

Expected result: the router loads the test preset, starts the model instance, initializes MTP, and listens on port 8080.

## 4. Test the API without thinking

```
curl -s http://127.0.0.1:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "test",
    "messages": [
      {"role": "user", "content": "Write one short sentence about Linux."}
    ],
    "chat_template_kwargs": {
      "enable_thinking": false
    },
    "max_tokens": 200
  }' | jq
```

## 5. Monitor RAM, GPU load and GPU memory

System RAM and swap:

```
watch -n 1 free -h
```

AMD GPU activity:

```
radeontop
```

GPU VRAM and GTT usage from the kernel:

```
watch -n 1 'printf "VRAM used: "; numfmt --to=iec < /sys/class/drm/card1/device/mem_info_vram_used; printf "VRAM total: "; numfmt --to=iec < /sys/class/drm/card1/device/mem_info_vram_total; printf "GTT used:  "; numfmt --to=iec < /sys/class/drm/card1/device/mem_info_gtt_used; printf "GTT total: "; numfmt --to=iec < /sys/class/drm/card1/device/mem_info_gtt_total'
```

## 6. Increase the AMDGPU GTT budget

The default GTT limit on this system was roughly half of usable RAM. The measured values were:

```
cat /sys/module/ttm/parameters/pages_limit
getconf PAGE_SIZE
```

The original values were 3475342 pages with 4096-byte pages, giving roughly 13.3 GiB of TTM/GTT budget. For this machine, an 18 GiB target was tested successfully:

```
4718592
```

Edit GRUB:

```
sudo nano /etc/default/grub
```

Set the kernel command line to:

```
GRUB_CMDLINE_LINUX_DEFAULT="ttm.pages_limit=4718592"
```

Then apply and reboot:

```
sudo update-grub
sudo reboot
```

After reboot, verify the new value:

```
cat /sys/module/ttm/parameters/pages_limit
```

It should return:

```
4718592
```

Then check the new GTT ceiling with:

```
radeontop
```

Observed result on this system: the GTT ceiling increased to roughly 18.4 GiB, the model ran stably, and generation improved by about 3 tokens/s compared with the lower GTT limit.

## 7. Final load-mode choice

Both load modes were tested after increasing GTT. load-mode=none could run with the larger budget, but did not provide a useful speed advantage. mmap remained stable and was retained.

```
load-mode = mmap
```

The warning about CPU tensor overrides with mmap can be accepted for this tested configuration because the measured throughput was not worse and mmap provided the more reliable path during earlier RADV allocation failures.

## 8. Fill most of the 128K context for a stress test

This sends a large synthetic prompt while leaving some headroom below the 131072-token limit:

```
python3 - <<'PY' | curl -s http://127.0.0.1:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  --data-binary @- | jq
import json

text = ("Linux kernel memory management and GPU scheduling. " * 15500)

payload = {
    "model": "test",
    "messages": [
        {"role": "user", "content": text}
    ],
    "chat_template_kwargs": {
        "enable_thinking": False
    },
    "max_tokens": 32
}

print(json.dumps(payload))
PY
```

## 9. Current known-good tuning baseline

| Setting | Current value |
| --- | --- |
| Model | Qwen3.6-35B-A3B-UD-IQ4_NL.gguf |
| Backend | Vulkan / RADV on Radeon 890M |
| Context | 131072 (128K) |
| Concurrency | 1 |
| Main KV cache | Q8_0 K + Q8_0 V |
| MTP | Enabled |
| MTP draft length | 3 |
| MTP KV cache | Q8_0 K + Q8_0 V |
| Flash attention | On |
| Load mode | mmap |
| GPU layers | Automatic |
| CPU thread overrides | None |
| TTM/GTT page limit | 4718592 pages (~18 GiB target) |
| Observed generation | About 32 tokens/s under the tested workload |

## 10. What is not configured yet

Autostart is intentionally not part of this stage. The server is still started manually. A systemd service should be added only after the final production model and production runtime settings are chosen and tested.
