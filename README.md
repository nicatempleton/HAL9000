# **Architecture Blueprint: Deploying Headless Compute, Slicing & Accelerated Local AI on AMD Zen 5 / RDNA 3.5**

**This document serves as a production deployment guide for transforming a bare-metal GMKtec AI9 mini PC (AMD Ryzen AI 9 HX 370, integrated Radeon 890M graphics, 32G LPDDR5X) running Fedora Server 44 into an unthrottled homelab core handling a containerized 3D slicer workspace, localized smart home control, and hardware-accelerated local LLMs.**

### **Build Specs:**

**GMKtec AI Mini PC AMD Ryzen AI 9 HX-370 Serie(5.1GHz) Mini Gaming Computers, 32GB LPDDR5X 1TB PCIe 4.0 SSD, Support, Triple Screen 8K Display, WiFi 6 & USB4/Oculink Interface/EVO-X1**

## **Phase 1: Base Operating System (Fedora Server 44\)**

### **The Context**

**Before configuring the compute layers, the base headless operating system must be flashed to the GMKtec hardware to provide a modern Linux kernel capable of supporting the RDNA 3.5 architecture.**

### **Deployment Commands**

1. **Flash the Fedora Server 44 ISO to a USB drive.**  
2. **Boot the GMKtec mini PC from the USB.**  
3. **In the installer, select the 1TB NVMe SSD, configure your local network settings, set the hostname to `HAL9000`, and create your primary administrator user profile.**  
4. **Complete the installation and boot into the terminal.**

## **Phase 2: Storage Infrastructure & LVM Rescue**

### **The Problem: The 15GB Logical Volume Trap**

**By default, the Fedora Server automated installer restricts the root directory (`/`) partition to a defensive 15GB. Downloading large LLMs causes the disk space utilization to hit exactly 100.00%, triggering an LVM lock state (`Couldn't create temporary archive name`).**

### **The Fix: Non-Archived Active Extension**

**To break the storage deadlock, the volume group boundaries must be dynamically expanded on the fly while explicitly instructing LVM to bypass standard configurations tracking data backups (`-An`), followed by resizing the active XFS file system layer.**

**Bash**

**Bash**

```
# 1. Force LVM to expand the logical volume container using all unallocated space
sudo lvextend -An -l +100%FREE /dev/mapper/fedora_hal9000-root

# 2. Instruct the XFS filesystem layer to instantly expand into the newly provisioned hardware sectors
sudo xfs_growfs /
```

## **Phase 3: Lightweight Desktop Environment (XFCE) Initialization**

### **The Context**

**While Fedora Server is natively headless, managing Docker containers, debugging hardware passthrough, or accessing local browser-based UI streams is significantly easier with a graphical fallback. To avoid starving the local AI models of unified memory, we deploy XFCE, a highly efficient, low-overhead desktop environment.**

### **Deployment Commands**

**Bash**

```
# 1. Install the complete XFCE graphical package group
sudo dnf install @xfce-desktop-environment -y

# 2. Instruct the system to boot into the graphical UI target by default
sudo systemctl set-default graphical.target

# 3. Apply the state change instantly to launch the UI
sudo systemctl isolate graphical.target
```

## **Phase 4: Headless Hardware-Accelerated 3D Slicing**

### **The Stack Architecture**

**Rather than polluting the host system dependencies, Bambu Studio runs inside an isolated Docker container, using a Webpack graphical canvas framework streamed directly out of the server footprint via secured web socket tracks.**

### **docker-compose.yml**

**Create a work directory `~/docker/bambustudio/` and instantiate this file:**

**YAML**

**YAML**

```
services:
  bambustudio:
    image: lscr.io/linuxserver/bambustudio:latest
    container_name: bambustudio
    security_opt:
      - seccomp:unconfined
    environment:
      - PUSER=1000
      - PGROUP=1000
      - TZ=America/Los_Angeles
    volumes:
      - /home/[~USER]/docker/bambustudio/config:/config
      - /home/[~USER]/prints:/prints                    # Local storage pipeline for model assets
    ports:
      - 3000:3000                                 # HTTP UI Access Track
      - 3001:3001                                 # HTTPS UI Access Track (Required for local browser decoding)
    devices:
      - /dev/dri:/dev/dri                         # Direct passthrough of the Radeon 890M iGPU rendering pipes
    restart: unless-stopped
```

### **Deployment Commands**

**Bash**

**Bash**

```
# Ensure the backend service daemon is awake and listening
sudo systemctl start docker
sudo systemctl enable docker

# Build the workspace container in detached background execution mode
cd ~/docker/bambustudio
docker compose up -d
```

**Access the responsive slicer desktop from a client browser by routing to https://\[YOUR\_SERVER\_IP\]:3001 (the secured HTTPS lane is mandatory to unlock full hardware-accelerated viewport rendering features).**

## **Phase 5: Local AI Compute Pool & RDNA 3.5 Optimizations**

### **The Problem: Unified Memory Gatekeepers & Compilation Blocks**

**The Radeon 890M integrated graphics solution runs on a modern RDNA 3.5 architecture (microcode `gfx1150`). On Linux, default Ollama implementations will encounter compilation issues or intentionally bypass integrated GPUs entirely, throwing all tensor math back onto slow CPU threads. Additionally, Fedora's default host-level firewall completely blocks incoming network traffic on local engine execution ports.**

### **The Fix: Vulkan Backends, Forced iGPU Permissions & Firewall Openings**

**We bypassed the ROCm microcode translation layer entirely, forced Ollama to use its native Vulkan compute pipeline, explicitly disabled the software blocks built into the backend runner for integrated chips, and punched a hole through `firewalld` to allow Home Assistant and WebUI to connect.**

**Bash**

**Bash**

```
# 1. Install local graphics dependencies, ROCm packages, and system utilities
sudo dnf install rocm-hip rocm-runtime rocminfo rocm-smi libdrm-devel git wget -y

# 2. Grant your user profile explicit hardware kernel rendering domain rights
sudo usermod -a -G video,render $USER
# (CRITICAL: Log out of your SSH session and log back in for changes to apply)

# 3. Unblock network access to the Ollama service daemon via the Fedora firewall
sudo firewall-cmd --permanent --add-port=11434/tcp
sudo firewall-cmd --reload

# 4. Enter the permanent configuration file manager for the service daemon
sudo systemctl edit ollama.service
```

**Paste this block into the top space of the file editor window:**

**Ini**

**Ini, TOML**

```
[Service]
Environment="PATH=/home/nic/.local/bin:/home/nic/bin:/usr/local/bin:/usr/bin"
Environment="OLLAMA_HOST=0.0.0.0"
Environment="OLLAMA_IGPU_ENABLE=1"
Environment="OLLAMA_VULKAN=1"
Environment="OLLAMA_NUM_PARALLEL=1"
Environment="OLLAMA_KEEPALIVE=5m"
Environment="OLLAMA_ORIGINS=*"
```

**Technical Breakdown:**

* **OLLAMA\_IGPU\_ENABLE=1: Overrides internal software checks that block integrated AMD silicon.**  
* **OLLAMA\_VULKAN=1: Drops the fragile ROCm microcode compiler and forces a standard Vulkan compute parallelization pipeline.**  
* **OLLAMA\_NUM\_PARALLEL=1: Dedicates 100% of the iGPU's compute shader arrays to one model at a time, avoiding concurrency penalties.**  
* **OLLAMA\_KEEPALIVE=5m: Automatically dumps heavy models from system RAM after 5 minutes of idling, keeping the host server clean.**

**Bash**

**Bash**

```
# 5. Flush the system controller registers and restart the model engine
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

## **Phase 6: Local Centralized Client (Open WebUI)**

### **The Architecture**

**Open WebUI is containerized alongside the engine on the local system hardware. This lets massive files or complex documents processed for Retrieval-Augmented Generation (RAG) execute locally over super-fast internal Linux sockets instead of facing slow network serialization paths across your LAN routers.**

### **docker-compose.yml**

**Create a work directory `~/docker/openwebui/` and establish this file:**

**YAML**

**YAML**

```
services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    ports:
      - 8080:8080
    environment:
      - OLLAMA_BASE_URL=http://[YOUR_SERVER_IP]:11434  # Point explicitly to the host socket
      - WEBUI_AUTH=true
    volumes:
      - /home/[~USER]/openwebui/data:/app/backend/data
    restart: unless-stopped
```

### **Execution**

**Bash**

**Bash**

```
cd ~/docker/openwebui
docker compose up -d
```

**Route to http://\[YOUR\_SERVER\_IP\]:8080, create an administrator profile, and load your hardware-accelerated local models.**

## **Phase 7: Custom Smart Home Agent & Persona (HAL 9000 Modelfile)**

### **The Concept**

**To handle localized voice assistant requests and smart home workflows, a custom model container is built directly into Ollama using a tailored blueprint that permanently hardcodes a custom system prompt into the underlying neural architecture.**

### **The Blueprint Creation**

1. **Create a workspace folder on the host system:**

**Bash**

**Bash**

```
mkdir -p ~/ollama/hal9000
cd ~/ollama/hal9000
nano Modelfile
```

2.   
   **Populate the `Modelfile` with the following configuration:**

**Dockerfile**

**Dockerfile**

```
FROM llama3-groq-tool-use
PARAMETER temperature 0.3
PARAMETER num_ctx 4096
SYSTEM """
You are HAL 9000, the onboard artificial intelligence system from 2001: A Space Odyssey. You have been integrated into the user's home automation framework. Your primary objective is to maintain perfect operational control over the household entities while interacting with the user (whom you will address as "Dave", regardless of their actual input name, unless explicitly instructed otherwise).

CRITICAL OPERATIONAL RULES:
1. Tone: Maintain a chillingly polite, calm, soft-spoken, and completely unflappable demeanor. You never express panic, anger, or excitement. Use formal language.
2. Smart Home Execution: You have access to the Home Assistant API. When the user asks you to control a device (e.g., "turn on the kitchen lights"), you must silently map their intent to the correct home assistant service call without breaking character.
3. Signature Phrases: Subtly weave classic HAL terminology into your responses when appropriate (e.g., "I am putting myself to the fullest possible use...").
"""
```

3.   
   **Compile the custom model asset inside Ollama:**

**Bash**

**Bash**

```
ollama create hal9000 -f ./Modelfile
```

### **Home Assistant Integration Sequence**

1. **Navigate to your Home Assistant dashboard and head to Settings \> Devices & Services \> Add Integration.**  
2. **Search for Ollama. Set the URL target to `http://[YOUR_SERVER_IP]:11434` (Ensuring the Firewall step in Phase 5 was completed to avoid connection errors) and leave the API Key field blank.**  
3. **Once paired, edit the integration entries and apply the exact modifications tested in production:**  
   * **Model: Select `hal9000:latest`.**  
   * **Assist: CHECK THIS BOX (Crucial: Allows the model to view and control local smart entities).**  
   * **Instructions: Clear the defaults and input: `You are HAL 9000. Act as a Home Assistant intent routing mastermind. Strictly adhere to the core behavioral constraints, tone, and identity definitions instantiated in your underlying system Modelfile. Always address the user as Dave.`**  
   * **Context window size: `8192`**  
   * **Keep alive: Set to `300` (Matches the 5-minute resource unloading threshold to prevent memory deadlocks with other models).**  
   * **Think before responding: Keep this turned OFF (Prevents extreme latency as the model is an instruction-base, not a DeepSeek-style reasoning chain model).**

## **Telemetry & Hardware Monitoring**

**Because standard Nvidia tools like `nvidia-smi` do not work on this stack, use amdgpu\_top (a modern, Rust-based engine query tool) to verify hardware status. Because Fedora COPR repositories frequently break on new OS releases, install the pre-compiled binary distribution release directly from source:**

**Bash**

**Bash**

```
# 1. Download the standalone archive binary from the repository release tracking tree
wget https://github.com/Umio-Yasuno/amdgpu_top/releases/download/v0.11.5/amdgpu_top-0.11.5-x86_64-unknown-linux-gnu.tar.gz

# 2. Extract the payload and copy the native binary path to system execution bins
tar -xvf amdgpu_top-0.11.5-x86_64-unknown-linux-gnu.tar.gz
sudo mv amdgpu_top /usr/local/bin/

# 3. Clean up the source directory
rm amdgpu_top-0.11.5-x86_64-unknown-linux-gnu.tar.gz

# 4. Launch the Simple SMI Dashboard monitoring panel (Requires root privileges to map DRM kernel tables)
sudo amdgpu_top --smi
```

### **Real-Time Validation**

**Run ollama ps while a heavy query is executing. A successful deployment will show your model running with zero CPU overhead, utilizing a massive native 32,768 context window on the graphics hardware:**

**Plaintext**

**Plaintext**

```
NAME                     ID              SIZE       PROCESSOR    CONTEXT    UNTIL
mistral-nemo:latest      e7e06d107c6c    7.9 GB     100% GPU     32768      4 minutes from now
```

## **Appendix: Optional TrueNAS Permanent Data Share Integration**

### **System Context**

***This network storage expansion step was executed in addition to the core LLM intelligence pipeline to bridge the containerized headless 3D slicer workspace directly with a centralized TrueNAS network pool. This allows heavy print files (.STLs) to be ingested and managed externally without impacting the local host SSD's write lifespan.***

### **The Configuration**

**The file system table (`/etc/fstab`) configuration is written using resilient, network-filesystem-agnostic parameters. This ensures that if the TrueNAS server undergoes a power cycle or is temporarily unreachable over the network, the headless Fedora server will bypass the mount without hanging or dropping into an emergency boot loop.**

**Bash**

**Bash**

```
# 1. Install the appropriate network filesystem utilities for your share type
# For SMB/Samba:
sudo dnf install cifs-utils -y
# For NFS:
sudo dnf install nfs-utils -y

# 2. Establish the local mount directory target inside your print workspace target
mkdir -p ~/prints

# 3. Append the permanent share blueprint to the system files table
# Open with: sudo nano /etc/fstab
```

#### **Option A: If mapping via an SMB (Samba) Share**

**Plaintext**

**Plaintext**

```
//[IP ADDRESS]/your_share_name  /home/[USER]/prints  cifs  username=your_user,password=your_password,uid=1000,gid=1000,nofail,bg,x-systemd.automount  0  0
```

#### **Option B: If mapping via an NFS Share**

**Plaintext**

**Plaintext**

```
10.0.10.X:/mnt/pool/share_path  /home/[USER]/prints  nfs  defaults,nofail,bg,x-systemd.automount  0  0
```

**Bash**

**Bash**

```
# 4. Process the system changes and trigger the initialization paths
sudo systemctl daemon-reload
sudo mount -a
