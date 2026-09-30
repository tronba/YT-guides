# llama.cpp Autostart with systemd

Ubuntu Server 26.04 · Vulkan / AMD iGPU · dedicated llama service user

Starting point. llama.cpp is already built under /opt/llama.cpp, the model is under /opt/llama-models, the tested runtime configuration is /opt/llama.cpp/llama-server.conf, and the dedicated service account is llama.

## 1. Create the systemd service

Create the service unit:

```
sudo nano /etc/systemd/system/llama-server.service
```

Enter:

```
[Unit]
Description=llama.cpp server
After=network.target

[Service]
Type=simple
User=llama
Group=llama
SupplementaryGroups=render video
WorkingDirectory=/opt/llama.cpp

Environment=HOME=/var/lib/llama
Environment=XDG_CACHE_HOME=/var/lib/llama/.cache

ExecStartPre=/bin/sleep 15

ExecStart=/opt/llama.cpp/build/bin/llama-server \
    --models-preset /opt/llama.cpp/llama-server.conf \
    --models-max 1 \
    --host 0.0.0.0 \
    --port 8080

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

## 2. Prepare the service user cache directory

The Vulkan shader cache needs a writable home/cache location for the llama service user:

```
sudo mkdir -p /var/lib/llama/.cache
sudo chown -R llama:llama /var/lib/llama
```

## 3. Load the unit and enable autostart

```
sudo systemctl daemon-reload
sudo systemctl enable --now llama-server
```

This starts the server immediately and enables it for future boots. The 15-second pre-start delay gives the AMDGPU/RADV stack time to become ready before llama.cpp initializes its Vulkan backend.

## 4. Verify the service

```
systemctl status llama-server
```

For the current boot log:

```
journalctl -u llama-server -b --no-pager
```

For live logs:

```
journalctl -u llama-server -f
```

## 5. Verify the API

Check that the router exposes the configured model:

```
curl -s http://127.0.0.1:8080/v1/models | jq
```

Then send a short chat request:

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

## 6. Verify after a reboot

```
sudo reboot
```

After reconnecting:

```
systemctl status llama-server
curl -s http://127.0.0.1:8080/v1/models | jq
```

## What “loaded” means with mmap

With load-mode = mmap, the GGUF remains file-backed and is mapped into the server process rather than eagerly copied into anonymous memory. Do not use radeontop memory usage alone to decide whether the model is loaded or whether Vulkan is active; verify actual inference performance and the service logs.

On this server, starting llama.cpp too early during boot caused it to run CPU-only even though the model loaded and answered requests. A 15-second ExecStartPre delay was tested successfully so the Vulkan backend initializes after the AMD GPU stack is ready.

## Useful service commands

```
sudo systemctl restart llama-server
sudo systemctl stop llama-server
sudo systemctl start llama-server
sudo systemctl disable llama-server
```
