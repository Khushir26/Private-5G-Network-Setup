# 📡 Private 5G Network Deployment — Open5GS + UERANSIM + srsRAN

A complete, beginner-friendly guide to build and run your own Private 5G Network from scratch using open-source tools.  
No prior 5G experience needed — every step is explained in plain language.

---

## 📌 Table of Contents

1. [What is a Private 5G Network?](#-what-is-a-private-5g-network)
2. [How This Guide is Organized](#-how-this-guide-is-organized)
3. [Understanding the Key Components](#-understanding-the-key-components)
4. [Architecture Overview](#-architecture-overview)
5. [System Requirements](#-system-requirements)
6. [Before You Begin](#-before-you-begin--important-notes)
7. [Part 1: Open5GS + UERANSIM — Software-Based Setup](#-part-1-open5gs--ueransim--software-based-setup)
   - [VM1 — Installing and Configuring the 5G Core](#vm1--installing-and-configuring-the-5g-core)
   - [VM2 — Installing and Running UERANSIM](#vm2--installing-and-running-ueransim)
   - [Connecting Everything and Running the Network](#connecting-everything-and-running-the-network)
8. [Part 2: Open5GS + srsRAN — Hardware-Based Setup](#-part-2-open5gs--srsran--hardware-based-setup)
9. [Testing and Verifying Your Network](#-testing-and-verifying-your-network)
10. [Understanding What Happened](#-understanding-what-happened)
11. [Troubleshooting](#-troubleshooting)
12. [Frequently Asked Questions](#-frequently-asked-questions)
13. [References](#-references)
14. [Disclaimer](#-disclaimer)

---

## 🌐 What is a Private 5G Network?

A **Private 5G Network** is a dedicated cellular network that you own and control entirely — it is not shared with the public. Think of it like having your own private Wi-Fi network, but built on the same technology that your mobile phone uses for 5G.

In a regular public 5G network, a telecom company owns the towers and controls everything. In a private 5G network, **you build and operate the network yourself**, for your own use.

**Why build one?**
- **Complete Control** — You decide who connects and what policies apply
- **High Privacy** — Your data never leaves your own infrastructure
- **No Telecom Dependency** — No monthly carrier bills or external limitations
- **Research and Learning** — Understand real 5G from the inside out

This guide uses **100% free, open-source software** — no expensive licenses required. You will set up a working 5G network on regular Linux machines, perfect for learning and research.

---

## 📖 How This Guide is Organized

| Approach | Tools Used | Best For | Hardware Needed |
|----------|-----------|----------|----------------|
| **Part 1 — Software Only** | Open5GS + UERANSIM | Learning, testing, research | Just 2 Linux VMs |
| **Part 2 — Real Hardware** | Open5GS + srsRAN + USRP B210 | Real RF testing, lab setups | USRP B210 SDR + programmable SIM |

If you are new to 5G, **start with Part 1**. It requires no special hardware.

Part 2 builds on top of Part 1 — the 5G Core (Open5GS) setup is identical. Only the RAN (radio) side changes.

---

## 🧠 Understanding the Key Components

### Open5GS — The 5G Core Network

Open5GS is an open-source implementation of the 5G Core. It is the "brain" of the network — handling authentication, session management, and routing data to the internet.

It runs as a collection of background services, each responsible for one task:

| Network Function | Full Name | What It Does |
|-----------------|-----------|-------------|
| **AMF** | Access and Mobility Management Function | Handles UE registration — like a receptionist checking your ID |
| **SMF** | Session Management Function | Manages data sessions — routes your connection |
| **UPF** | User Plane Function | Forwards actual data packets to the internet |
| **NRF** | Network Repository Function | Directory of all running services |
| **AUSF** | Authentication Server Function | Verifies user identity |
| **UDM** | Unified Data Management | Stores subscriber information |
| **PCF** | Policy Control Function | Applies network rules and limits |

Open5GS implements the full 3GPP Release 17 compliant 5G Core, running as **17 background services** 
(plus a WebUI), covering all mandatory Network Functions including AMF, SMF, UPF, NRF, AUSF, UDM, 
PCF, UDR, NSSF, BSF, SCP, and SEPP. The table above covers the 7 functions you will directly 
configure or monitor in this guide.

### UERANSIM — Simulated Base Station and Device

UERANSIM simulates two things in software:
- **gNB (gNodeB)** — the 5G base station (the "tower")
- **UE (User Equipment)** — the 5G device (like a smartphone)

In real life these are physical devices. UERANSIM lets you test the full 5G flow with no physical radio hardware.

### srsRAN — Real Radio Software (Part 2)

srsRAN is a real 5G RAN stack. Unlike UERANSIM which simulates radio, srsRAN actually transmits and receives real 5G radio signals through a USRP B210 hardware device.

### MongoDB — The Database

MongoDB stores all subscriber data for Open5GS — IMSI numbers, authentication keys, and session info. It must be running before Open5GS starts.

### USRP B210 — Software Defined Radio Hardware (Part 2 only)

The USRP B210 is an SDR (Software Defined Radio) — hardware that transmits and receives radio signals across a wide frequency range, fully controlled by software.

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                    Public Internet / WAN                          │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                     ┌───────▼───────┐
                     │   NAT / GW    │  ← VM1 NATs UE traffic to internet
                     └───────┬───────┘
                             │
            ┌────────────────┼─────────────────┐
            │                                   │
┌───────────▼─────────────┐       ┌────────────▼────────────┐
│      VM1 — 5G Core      │       │    VM2 — RAN / UE       │
│                         │       │                         │
│  Open5GS Services:      │       │  UERANSIM:              │
│  ├── AMF  (port 38412)  │       │  ├── nr-gnb (gNodeB)    │
│  ├── SMF               │◄──N2──►│  └── nr-ue  (UE)        │
│  ├── UPF  (port 2152)  │◄──N3──►│                         │
│  ├── NRF               │       │  OR  srsRAN:             │
│  ├── AUSF, UDM, PCF    │       │  └── gnb (real RF)       │
│  ├── MongoDB (27017)    │       │                         │
│  └── WebUI (port 9999) │       │                         │
└─────────────────────────┘       └─────────────────────────┘

N2 = Control Plane (NGAP/SCTP, port 38412)
N3 = User Plane  (GTP-U/UDP, port 2152)
```

**In plain terms:** VM1 is the server running all 5G network logic. VM2 acts as both the phone (UE) and tower (gNB). They communicate over your local network. VM1 also connects to the internet so the UE can browse through the 5G core.

---

## 💻 System Requirements

### Part 1 — Software Setup

You need two machines (physical or virtual) that can ping each other.

| Item | VM1 (5G Core) | VM2 (RAN/UE) |
|------|--------------|-------------|
| OS | Ubuntu 22.04 LTS | Ubuntu 22.04 LTS |
| RAM | 4 GB min, 8 GB recommended | 2 GB min, 4 GB recommended |
| Storage | 20 GB | 10 GB |
| CPU | 2 cores | 2 cores |
| Network | Must reach VM2 and internet | Must reach VM1 |

> Using VirtualBox or VMware? Set both network adapters to **Bridged Mode** so the VMs get IPs on your local network and can reach each other.

### Part 2 — Additional Hardware

| Item | Specification |
|------|-------------|
| USRP B210 SDR | Connected via **USB 3.0** (USB 2.0 will not work) |
| Programmable SIM Card | Blank programmable SIM (from Open-Cells or sysmocom) |
| SIM Card Reader | USB smart card reader |
| RAM | 16 GB recommended for srsRAN |
| CPU | Intel i7 or equivalent (srsRAN is CPU-intensive) |

---

## ⚠️ Before You Begin — Important Notes

Please read these before starting:

1. **Find your VM IPs first.** On each VM, run `ip addr show` and note the IP of the main network interface (usually `eth0` or `ens33`). Replace `<VM1-IP>` and `<VM2-IP>` throughout this guide with the actual IPs.

2. **VMs must ping each other.** Test before you start:
   ```bash
   # Run on VM2
   ping <VM1-IP>
   ```
   If ping fails, fix your network before proceeding.

3. **Run commands as your regular user with `sudo` where shown.** Do not run everything as root.

4. **Do steps in order.** Each step builds on the previous one.

5. **PLMN values used in this guide:** MCC=`999`, MNC=`70` (PLMN `99970`) — these are reserved test network codes for lab use only.

6. **Estimated time:** Part 1 takes approximately 2–3 hours for a first-time setup.

---

## 🔧 Part 1: Open5GS + UERANSIM — Software-Based Setup

---

### VM1 — Installing and Configuring the 5G Core

> All commands in this section are run on **VM1**.

---

#### Step 1: Install MongoDB

MongoDB is the database Open5GS uses to store subscriber profiles and session data. It must be running before Open5GS starts.

We install MongoDB 8.0 from the official MongoDB repository (Ubuntu's default version is too old).

```bash
# Update system packages
sudo apt update

# Install tools needed to add the MongoDB repository
sudo apt install -y gnupg curl

# Add MongoDB's official GPG signing key
curl -fsSL https://pgp.mongodb.com/server-8.0.asc | \
  sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg --dearmor

# Add the MongoDB 8.0 repository for Ubuntu 22.04
echo "deb [arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg] \
  https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/8.0 multiverse" | \
  sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list

# Refresh package list
sudo apt update

# Install MongoDB
sudo apt install -y mongodb-org

# Start MongoDB now and enable it to start on every reboot
sudo systemctl start mongod
sudo systemctl enable mongod
```

**Verify MongoDB is running:**

```bash
sudo systemctl status mongod
```

Look for `Active: active (running)` in green. If not, check logs:

```bash
sudo journalctl -u mongod -n 50
```

---

#### Step 2: Install Open5GS

Open5GS is available from its own Ubuntu PPA, making installation straightforward.

```bash
# Install software-properties-common (needed for add-apt-repository)
sudo apt install -y software-properties-common

# Add the Open5GS PPA
sudo add-apt-repository ppa:open5gs/latest

# Update package list
sudo apt update

# Install Open5GS — installs all 5G core network functions at once
sudo apt install -y open5gs
```

Open5GS automatically starts all its services after installation.

**Check all services started:**

```bash
sudo systemctl status open5gs-*
```

Every service should show `active (running)`. Open5GS installs **18 services** in total.

**Quick count:**

```bash
sudo systemctl status open5gs-* | grep -c "active (running)"
```

You should see **17** at this point — the 18th (WebUI) is added in the next step. If the count is lower, check which service failed:

```bash
sudo journalctl -u open5gs-amfd -n 30   # Replace amfd with the failing service name
sudo systemctl restart open5gs-amfd
```

---

#### Step 3: Install the Open5GS WebUI

The WebUI is a web dashboard for managing subscribers (adding/editing UE devices). It runs on port `9999` and requires Node.js.

```bash
# Install prerequisites
sudo apt update
sudo apt install -y ca-certificates curl gnupg

# Create keyring directory
sudo mkdir -p /etc/apt/keyrings

# Add Node.js 20.x signing key and repository
curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg

NODE_MAJOR=20
echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] \
  https://deb.nodesource.com/node_$NODE_MAJOR.x nodistro main" | \
  sudo tee /etc/apt/sources.list.d/nodesource.list

# Install Node.js
sudo apt update
sudo apt install -y nodejs

# Verify Node.js installed
node --version
# Expected: v20.x.x

# Install the Open5GS WebUI
curl -fsSL https://open5gs.org/open5gs/assets/webui/install | sudo -E bash -
```

**Access the WebUI:**

Open a browser and go to `http://<VM1-IP>:9999`

Default login credentials:

| Field | Value |
|-------|-------|
| Username | `admin` |
| Password | `1423` |

> ⚠️ **Change the default password after your first login.** These credentials are publicly known.

---

#### Step 4: Configure AMF (Access and Mobility Management Function)

By default, Open5GS binds to `127.x.x.x` (localhost). Since our gNB is on VM2, we must configure the AMF to listen on VM1's actual network IP.

```bash
sudo nano /etc/open5gs/amf.yaml
```

Find and update the relevant sections as shown below. Only change the IP addresses — leave all other values as they are.

```yaml
amf:
  sbi:
    server:
      - address: <VM1-IP>       # Change from 127.0.0.5 to your VM1 IP
        port: 7777
    client:
      scp:
        - uri: http://127.0.0.200:7777
  ngap:
    server:
      - address: <VM1-IP>       # Change from 127.0.0.5 — gNB connects here
  metrics:
    server:
      - address: <VM1-IP>       # Change from 127.0.0.5
        port: 9090
  guami:
    - plmn_id:
        mcc: 999
        mnc: 70
      amf_id:
        region: 2
        set: 1
  tai:
    - plmn_id:
        mcc: 999
        mnc: 70
      tac: 1                    # Tracking Area Code — must match gNB config
  plmn_support:
    - plmn_id:
        mcc: 999
        mnc: 70
      s_nssai:
        - sst: 1
  security:
    integrity_order: [ NIA2, NIA1, NIA0 ]
    ciphering_order: [ NEA0, NEA1, NEA2 ]
  network_name:
    full: Open5GS
  amf_name: open5gs-amf0
```

Save and exit: `Ctrl+X` → `Y` → `Enter`

**Restart AMF and verify:**

```bash
sudo systemctl restart open5gs-amfd
sudo systemctl status open5gs-amfd
```

**Check the AMF log:**

```bash
sudo tail -f /var/log/open5gs/amf.log
```

Press `Ctrl+C` to stop following the log.

---

#### Step 5: Configure UPF (User Plane Function)

The UPF forwards user data packets between the gNB and the internet.

```bash
sudo nano /etc/open5gs/upf.yaml
```

Update the `gtpu` server address to VM1's actual IP:

```yaml
upf:
  pfcp:
    server:
      - address: 127.0.0.7      # Leave as is — internal PFCP communication
    client:
      smf:
        - address: 127.0.0.4    # Leave as is — internal SMF address
  gtpu:
    server:
      - address: <VM1-IP>       # Change from 127.0.0.7 — gNB sends data here
  session:
    - subnet: 10.45.0.0/16
      gateway: 10.45.0.1
      dnn: internet             # Must match the DNN in the subscriber profile
  metrics:
    server:
      - address: 127.0.0.7
        port: 9090
```

**Restart UPF and verify:**

```bash
sudo systemctl restart open5gs-upfd
sudo systemctl status open5gs-upfd
```

**Check the UPF log:**

```bash
sudo tail -f /var/log/open5gs/upf.log
```

---

#### Step 6: Enable IP Forwarding and Configure NAT

When the UE browses the internet, traffic flows: **UE → gNB → UPF → Internet**. For this to work, VM1 needs to:
- **Forward packets** between network interfaces (IP forwarding)
- **NAT the UE's internal IP** to VM1's public IP before sending to the internet

Without these steps, the UE can connect but cannot reach the internet.

```bash
# Enable IP forwarding immediately
sudo sysctl -w net.ipv4.ip_forward=1

# Make it permanent across reboots
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf

# Add a NAT rule (auto-detects your outbound network interface)
sudo iptables -t nat -A POSTROUTING -o $(ip route show default | awk '/default/ {print $5}') -j MASQUERADE

# Allow all forwarded packets
sudo iptables -I FORWARD 1 -j ACCEPT

# Disable ufw — it can interfere with our iptables rules
sudo systemctl stop ufw
sudo systemctl disable ufw

# Save iptables rules so they persist after reboot
sudo apt install -y iptables-persistent
sudo dpkg-reconfigure iptables-persistent
# When prompted, select Yes to save both IPv4 and IPv6 rules
```

**Verify IP forwarding is active:**

```bash
cat /proc/sys/net/ipv4/ip_forward
# Must output: 1
```

---

#### Step 7: Add a Subscriber in the WebUI

Before the UE can connect, it must be registered in the 5G core database — equivalent to programming a SIM card.

1. Open your browser → URL: `http://localhost:9999` or `http://<VM1-IP>:9999`
2. Log in with `admin` / `1423`
3. Click **Subscribers** in the left menu
4. Click the **+** button to add a new subscriber
5. Fill in the following values:

| Field | Value |
|-------|-------|
| IMSI | `999700000000001` |
| Subscriber Key (K) | `465B5CE8B199B49FAA5F0A2EE238A6BC` |
| Operator Key Type | `OPc` |
| Operator Key (OPc) | `E8ED289DEBA952E4283B54E88E6183CA` |
| APN/DNN | `internet` |

6. Leave all other fields at default, then click **Save**

> ⚠️ **Critical:** These exact values — IMSI, K, and OPc — must match what you configure in the UE config file on VM2. One wrong character will cause authentication to fail silently.

---

**VM1 setup is complete. Move on to VM2.**

---

### VM2 — Installing and Running UERANSIM

> All commands in this section are run on **VM2**.

---

#### Step 8: Install Dependencies and Build UERANSIM

UERANSIM is built from source. The build takes a few minutes.

```bash
# Update and upgrade
sudo apt update
sudo apt upgrade -y

# Install build tools and required libraries
# libsctp-dev, lksctp-tools: for SCTP protocol (used on the N2 interface between gNB and AMF)
# iproute2: for managing TUN interfaces
sudo apt install -y make g++ libsctp-dev lksctp-tools iproute2

# Install cmake using snap (recommended method for UERANSIM)
sudo snap install cmake --classic

# Clone UERANSIM source code
git clone https://github.com/aligungr/UERANSIM
cd UERANSIM

# Build UERANSIM (takes approximately 3–5 minutes)
make
```

After the build completes, two executables will be in the `build/` folder:
- `build/nr-gnb` — simulated gNB (base station)
- `build/nr-ue` — simulated UE (mobile device)

---

#### Step 9: Configure the gNB (Base Station)

```bash
nano config/open5gs-gnb.yaml
```

Update the following fields (leave everything else unchanged):

```yaml
mcc: '999'           # Mobile Country Code — must match Open5GS AMF config
mnc: '70'            # Mobile Network Code — must match Open5GS AMF config

nci: '0x000000010'   # NR Cell Identity — unique ID for this cell
idLength: 32         # NCI bit length (leave as is)
tac: 1               # Tracking Area Code — must match TAC in Open5GS AMF config

linkIp: <VM2-IP>     # IP of this machine (VM2)
ngapIp: <VM2-IP>     # IP used for N2 control plane toward AMF
gtpIp: <VM2-IP>      # IP used for N3 user plane toward UPF

# Point to Open5GS AMF on VM1
amfConfigs:
  - address: <VM1-IP>
    port: 38412

# Required configuration field for supported network type
slices:
  - sst: 1

ignoreStreamIds: true   # Ignore SCTP stream errors (recommended)
```

> **Note:** The `tac` value here must match the `tac` under `tai` in `/etc/open5gs/amf.yaml` on VM1. Both are set to `1` in this guide.

---

#### Step 10: Configure the UE (User Equipment)

The UE config must exactly match the subscriber you registered in the WebUI.

```bash
nano config/open5gs-ue.yaml
```

Update these fields:

```yaml
# SUPI = IMSI with 'imsi-' prefix
supi: 'imsi-999700000000001'

mcc: '999'    # Must match gNB and AMF
mnc: '70'     # Must match gNB and AMF

# Must exactly match what you entered in the WebUI subscriber
key: '465B5CE8B199B49FAA5F0A2EE238A6BC'    # Subscriber Key (K)
op: 'E8ED289DEBA952E4283B54E88E6183CA'     # Operator Key (OPc value)
opType: 'OPC'    # Must be OPC to match the WebUI setting

amf: '8000'
imei: '356938035643803'

# The gNB this UE should connect to
gnbSearchList:
  - <VM2-IP>

# Data session — DNN must match the WebUI subscriber
sessions:
  - type: 'IPv4'
    apn: 'internet'
    slice:
      sst: 1
```

> ⚠️ **Double-check:** The `supi` (IMSI), `key`, and `op` in this file must be identical to the values in the Open5GS WebUI subscriber. Even one wrong character causes silent authentication failure.

---

### Connecting Everything and Running the Network

---

#### Step 11: Verify the Core is Running

On **VM1**, confirm all Open5GS services are active:

```bash
sudo systemctl status open5gs-*
```

If any service is not running:

```bash
sudo systemctl restart open5gs-*
```

---

#### Step 12: Start the gNB on VM2

Open a terminal on **VM2** and run:

```bash
cd ~/UERANSIM
./build/nr-gnb -c config/open5gs-gnb.yaml
```

**What to look for:**

```
[INFO] gNB is up and running
[INFO] SCTP connection established (AMF)
[INFO] NG Setup procedure is successful
```

The line **`NG Setup procedure is successful`** confirms the gNB connected to the AMF on VM1. If you see errors, check:
- The AMF on VM1 has the correct `<VM1-IP>` in `/etc/open5gs/amf.yaml`
- Port `38412` is reachable: `nc -zv <VM1-IP> 38412`
- TAC in gNB config matches TAC in AMF config

**Leave this terminal open.** The gNB must keep running.

---

#### Step 13: Start the UE on VM2

Open a **new terminal** on VM2 (keep the gNB terminal running):

```bash
cd ~/UERANSIM
sudo ./build/nr-ue -c config/open5gs-ue.yaml
```

> `sudo` is required because UERANSIM needs to create a TUN network interface.

**What to look for:**

```
[INFO] UE switches to state: MM-REGISTERED/NORMAL-SERVICE
[INFO] PDU Session Establishment successful
[INFO] TUN interface[uesimtun0, 10.45.0.x] is up
```

What these mean:
- **MM-REGISTERED** — UE successfully registered with the 5G core
- **PDU Session Establishment successful** — a data session was created
- **TUN interface[uesimtun0, ...]** — a virtual network interface now exists representing the UE's data connection

**Your 5G network is now fully operational! 🎉**

---

## 🔌 Part 2: Open5GS + srsRAN — Hardware-Based Setup

This part uses a real **USRP B210** to transmit actual 5G radio signals. The Open5GS Core (VM1) remains exactly the same — only the RAN part changes.

> **Prerequisites:** Complete the Open5GS core setup from Part 1 (Steps 1–7) first. You also need a USRP B210 connected via USB 3.0 and a programmable SIM card.

---

#### Step 1: Install srsRAN Build Dependencies

```bash
sudo apt update

sudo apt install -y cmake make gcc g++ pkg-config git \
  libfftw3-dev libmbedtls-dev libsctp-dev \
  libyaml-cpp-dev libgtest-dev libuhd-dev uhd-host
```

| Library | Purpose |
|---------|---------|
| `libfftw3-dev` | Fast Fourier Transform for signal processing |
| `libmbedtls-dev` | Cryptography for security functions |
| `libsctp-dev` | SCTP protocol for the N2 interface |
| `libyaml-cpp-dev` | YAML config file parsing |
| `libuhd-dev`, `uhd-host` | USRP Hardware Driver for the B210 |

---

#### Step 2: Download USRP B210 Firmware

The USRP B210 needs firmware images on the host machine before it can operate.

```bash
sudo python3 /usr/lib/uhd/utils/uhd_images_downloader.py
```

This downloads several hundred MB — wait for it to complete fully.

Now **connect your USRP B210 via USB 3.0** (the blue-colored USB port).

**Verify the device is detected:**

```bash
uhd_find_devices
```

Expected output:

```
-- UHD Device 0
--------------------------------------------------
Device Address:
    serial: <your-serial-number>
    product: B210
    type: b200
```

If not found, try a different USB 3.0 port. USB 2.0 will not work.

**Get full hardware info:**

```bash
uhd_usrp_probe
```

If this completes without errors, your hardware is working correctly.

---

#### Step 3: Clone and Build srsRAN

```bash
# Clone srsRAN Project source code
git clone https://github.com/srsRAN/srsRAN_Project.git
cd srsRAN_Project

# Create a build directory
mkdir build
cd build

# Generate build system
cmake ../

# Compile using all available CPU cores (takes 10–30 minutes)
make -j $(nproc)

# Optional: run test suite
make test -j $(nproc)

# Install to system paths
sudo make install
```

---

#### Step 4: Configure the srsRAN gNB

Navigate to the gNB binary directory:

```bash
cd srsRAN_Project/build/apps/gnb/
```

Copy the example config and edit your copy:

```bash
cp gnb_rf_b200_tdd_n78_20mhz.yml my_gnb.yml
sudo nano my_gnb.yml
```

Update the following:

```yaml
# AMF connection
cu_cp:
  amf:
    addr: <VM1-IP>          # IP address of Open5GS AMF
    port: 38412             # Standard N2 port
    bind_addr: <THIS-IP>    # IP of this machine (where srsRAN runs)
    supported_tracking_areas:
      - tac: 7              # Must match Open5GS AMF config
        plmn_list:
          - plmn: "99970"   # MCC(999) + MNC(70) — must match Open5GS

# Radio cell configuration
cell_cfg:
  dl_arfcn: 632628          # Downlink frequency (Band n78, ~3.5 GHz)
  band: 78                  # 5G NR Band 78
  channel_bandwidth_MHz: 20 # 20 MHz channel bandwidth
  common_scs: 30            # Subcarrier spacing 30 kHz
  plmn: "99970"
  tac: 7
  pci: 1                    # Physical Cell ID
```

> **Note:** Part 2 uses `tac: 7` — you must also update the `tac` under `tai` in `/etc/open5gs/amf.yaml` on VM1 to match.

---

#### Step 5: Program the Programmable SIM Card

A programmable SIM can be written with custom credentials to authenticate with your private network.

```bash
# Extract and compile the SIM programming tool
tar -xvzf uicc-v3.3.tgz
cd uicc-v3.3
make
```

Connect your USB SIM card reader with the programmable SIM inserted.

**Program the SIM:**

```bash
sudo ./program_uicc \
  --adm 12345678 \
  --imsi 999700000000001 \
  --isdn 00000001 \
  --acc 0001 \
  --key 6874736969202073796d4b2079650a73 \
  --opc 504f20634f6320504f50206363500a4f \
  --spn "PrivNet" \
  --authenticate \
  --noreadafter
```

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `--adm` | `12345678` | Admin unlock code for the SIM |
| `--imsi` | `999700000000001` | Subscriber identifier |
| `--key` | `6874736969...` | Authentication key (K) |
| `--opc` | `504f20634f...` | Operator key (OPc) |
| `--spn` | `PrivNet` | Network name shown on the device screen |

---

#### Step 6: Register the Hardware Subscriber in the WebUI

Go to `http://<VM1-IP>:9999` → **Subscribers** → **+** and add:

| Field | Value |
|-------|-------|
| IMSI | `999700000000001` |
| Subscriber Key (K) | `6874736969202073796d4b2079650a73` |
| Operator Key Type | `OPc` |
| Operator Key (OPc) | `504f20634f6320504f50206363500a4f` |
| DNN | `internet` |

> ⚠️ **These key values are different from the UERANSIM test keys in Part 1.** Use the exact values shown above for the hardware SIM — they must match what was programmed onto the SIM card.

---

#### Step 7: Run the Hardware 5G Network

**On VM1 — confirm the core is running:**

```bash
sudo systemctl status open5gs-*
# All services should show active (running)
```

**Start srsRAN gNB:**

```bash
cd srsRAN_Project/build/apps/gnb/
sudo ./gnb -c my_gnb.yml
```

Successful startup looks like:

```
--== srsRAN gNB (commit ...) ==--
[INFO] [UHD] B210 found
[INFO] Cell pci=1, bw=20 MHz
[INFO] Connected to AMF
```

Power on the UE device (phone or modem) with the programmed SIM inserted. It will automatically find your network (shown as "PrivNet") and register. You will see registration events appear on the gNB terminal in real time.

---

## ✅ Testing and Verifying Your Network

### Test 1: Confirm the TUN Interface Exists (Part 1)

On **VM2**, after the UE starts:

```bash
ip addr show
```

You should see `uesimtun0` with an IP address like `10.45.0.x`. This virtual interface represents your UE's data connection.

### Test 2: Ping Through the 5G Core

```bash
# Traffic goes: VM2 → gNB → UPF → Internet
ping -I uesimtun0 8.8.8.8
```

Expected output:

```
64 bytes from 8.8.8.8: icmp_seq=1 ttl=117 time=15.2 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=117 time=14.8 ms
```

If ping works, your entire 5G pipeline is functioning end-to-end. ✅

### Test 3: HTTP Request Through the 5G Interface

```bash
curl --interface uesimtun0 -X GET "https://httpbin.org/get"
```

This returns a JSON response. The IP shown will be VM1's public IP — because the UE's traffic is NATted through the UPF.

### Test 4: Throughput Test with iperf3

On a server reachable from the internet, start iperf3:

```bash
iperf3 -s
```

On VM2:

```bash
# Download test
iperf3 -c <server-ip> --bind-dev uesimtun0

# Upload test
iperf3 -c <server-ip> --bind-dev uesimtun0 -R
```

### Test 5: Monitor Core Logs in Real Time

```bash
# AMF log — shows UE registration and authentication events
sudo tail -f /var/log/open5gs/amf.log

# UPF log — shows data plane activity
sudo tail -f /var/log/open5gs/upf.log

# Or use journalctl
sudo journalctl -u open5gs-amfd -f
sudo journalctl -u open5gs-upfd -f
```

---

## 🧩 Understanding What Happened

When the UE showed `MM-REGISTERED`, here is the full sequence of events that occurred:

1. **UE → gNB:** UE sent a Registration Request over the simulated radio link
2. **gNB → AMF:** gNB forwarded it over NGAP (N2 interface) to the AMF on VM1
3. **AMF → AUSF/UDM:** AMF asked authentication functions to verify the UE's identity
4. **AUSF/UDM → AMF:** Authentication successful — keys in UE config matched the WebUI subscriber
5. **AMF → UE:** Registration Accepted — UE is now part of the network
6. **UE requests a data session:** UE sends a PDU Session Establishment Request
7. **SMF selects UPF:** SMF selects the UPF to handle this UE's traffic
8. **UPF creates a GTP tunnel:** A data tunnel is established between gNB and UPF
9. **UE receives an IP address:** UPF assigns the UE an IP (e.g., `10.45.0.2`)
10. **TUN interface appears on VM2:** `uesimtun0` is created with that IP
11. **Traffic flows:** Packets through `uesimtun0` travel the entire 5G pipeline to the internet

---

## 🔧 Troubleshooting

### MongoDB will not start

```bash
# Check last 50 log lines
sudo journalctl -u mongod -n 50

# Fix permission issues (most common cause)
sudo mkdir -p /var/lib/mongodb
sudo chown -R mongodb:mongodb /var/lib/mongodb

sudo systemctl restart mongod
```

### Open5GS services not starting

```bash
# Check individual service
sudo systemctl status open5gs-amfd
sudo journalctl -u open5gs-amfd -n 30

# Restart all services
sudo systemctl restart open5gs-*

# View real-time log
sudo tail -f /var/log/open5gs/amf.log
```

### gNB cannot connect to AMF — NG Setup fails

Work through these checks in order:

**1. Is the IP correct in gNB config?**
Open `config/open5gs-gnb.yaml` on VM2 — verify `address: <VM1-IP>` under `amfConfigs`

**2. Is the AMF listening on the right IP?**
Open `/etc/open5gs/amf.yaml` on VM1 — check `ngap → server → address` is VM1's actual IP, not `127.0.0.5`

**3. Can VM2 reach VM1?**
```bash
ping <VM1-IP>
```

**4. Is port 38412 reachable?**
```bash
# On VM1 — check AMF is listening
sudo ss -tulpn | grep 38412

# On VM2 — check you can reach it
nc -zv <VM1-IP> 38412
```

**5. TAC mismatch?**
The `tac` in `config/open5gs-gnb.yaml` must equal the `tac` under `tai` in `/etc/open5gs/amf.yaml`

**6. PLMN mismatch?**
The `mcc` and `mnc` in the gNB config must match the `plmn_id` values in the AMF config

### UE registration fails — authentication error

This almost always means the credentials do not match between the UE config and the WebUI subscriber.

1. Open `config/open5gs-ue.yaml` on VM2
2. Open the WebUI → Subscribers → find your subscriber
3. Compare **character by character**: `supi` (IMSI without the `imsi-` prefix), `key`, and `op`

Even one wrong digit causes complete authentication failure.

### UE registers but cannot reach the internet

```bash
# On VM1 — verify IP forwarding
cat /proc/sys/net/ipv4/ip_forward
# Must output: 1

# Check NAT rule is in place
sudo iptables -t nat -L POSTROUTING -n -v
# Should show a MASQUERADE rule

# Check ufw is disabled
sudo ufw status
# Should show: inactive
```

### USRP B210 not detected

```bash
# Check if OS sees the USB device
lsusb | grep Ettus
# Expected: Bus xxx Device xxx: ID 2500:0020 Ettus Research LLC USRP B210

# If seen in lsusb but not by UHD
sudo apt install --reinstall libuhd-dev uhd-host
sudo python3 /usr/lib/uhd/utils/uhd_images_downloader.py
```

Always use a **USB 3.0 port** (blue-colored). USB 2.0 does not provide sufficient bandwidth.

### srsRAN build fails

```bash
# Ensure all dependencies are present
sudo apt install -y cmake make gcc g++ pkg-config libfftw3-dev \
  libmbedtls-dev libsctp-dev libyaml-cpp-dev libgtest-dev libuhd-dev uhd-host

# Clean the build and start fresh
cd srsRAN_Project/build
make clean
cmake ../
make -j $(nproc)
```

---

## ❓ Frequently Asked Questions

**Q: Can I run both VMs on the same physical machine?**  
Yes. Use VirtualBox or VMware with both network adapters set to **Bridged Mode**. Your machine needs at least 8 GB of RAM total.

**Q: What if I only have one machine?**  
Open5GS and UERANSIM can run on the same machine using loopback addresses (`127.x.x.x`). The UERANSIM GitHub wiki has a dedicated single-machine setup guide.

**Q: Ping works but web browsing does not?**  
Browsing requires DNS. Test with a direct IP first:
```bash
curl --interface uesimtun0 http://8.8.8.8
```
If that works, configure DNS for the interface:
```bash
resolvectl dns uesimtun0 8.8.8.8
```

**Q: What is the difference between K and OPc?**  
`K` is the master secret key stored in the SIM. `OPc` is derived from `K` and the operator's key using a one-way function. Open5GS uses OPc by default — more secure than plain OP. Both must match identically between the UE/SIM and the network database.

**Q: How many UEs can I simulate with UERANSIM?**  
You can run multiple UEs by launching `nr-ue` multiple times, each with its own config file using a different IMSI. Each UE needs its own subscriber record in the WebUI. The limit is your machine's CPU and RAM.

**Q: Is this setup production-ready?**  
This is a lab and research setup. Before any real deployment: change all default credentials, configure strict firewall rules, restrict WebUI access to trusted IPs only, and set up regular MongoDB backups.

---

## 📚 References

- [Open5GS Official Documentation](https://open5gs.org/open5gs/docs/)
- [UERANSIM GitHub Repository](https://github.com/aligungr/UERANSIM)
- [srsRAN Project Documentation](https://docs.srsran.com/projects/project)
- [Open-Cells UICC Programming Tool](https://open-cells.com/index.php/uiccsim-programing/)
- [Ettus Research UHD Documentation](https://files.ettus.com/manual/)
- [3GPP TS 23.501 — 5G System Architecture](https://www.3gpp.org/ftp/Specs/archive/23_series/23.501/)

---

## 📄 Disclaimer

This project is developed entirely for **educational and research purposes**.

All testing was conducted in a controlled lab environment. Before deploying any radio transmitting equipment, ensure full compliance with your country's RF regulations and obtain all required licenses. Unauthorized RF transmission on licensed frequency bands is illegal in most jurisdictions.

The PLMN codes used in this guide (MCC=999, MNC=70) are reserved for test networks and are not valid on public networks.

---

> 💡 If this guide helped you, feel free to **star the repository** and share it with others learning 5G!
