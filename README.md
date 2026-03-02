# 📡 Private 5G Network Deployment
### Open5GS + UERANSIM + srsRAN

A step-by-step guide to deploy a fully functional private 5G network using open-source components — covering both software simulation and real hardware setups.

---

## 🌐 What is a Private 5G Network?

A **private 5G network** is a dedicated cellular network deployed within an organization's premises, fully owned and controlled by the operator. Unlike public carrier networks, a private 5G gives you:

- **Complete Control** — manage policies, QoS, and security yourself
- **Data Privacy** — traffic never leaves your own infrastructure
- **Low Latency** — optimized for local, time-sensitive applications
- **Network Slicing** — run multiple logical networks (eMBB, URLLC, mMTC) on the same infrastructure

This guide uses 100% free, open-source software. No licenses required.

---

## 🗺️ Deployment Approaches

| | Part 1 — Software Simulation | Part 2 — Hardware Testbed |
|---|---|---|
| **Stack** | Open5GS + UERANSIM | Open5GS + srsRAN + USRP B210 |
| **Best For** | Learning, dev, lab testing | Real RF testing, live deployment |
| **Hardware** | 2x Ubuntu VMs | USRP B210 SDR + programmable SIM |
| **Complexity** | Medium | High |
| **Start Here?** | ✅ Yes | After completing Part 1 |

---

## 🧠 Core Components

| Component | Role |
|---|---|
| **Open5GS** | Open-source 5G Core — runs AMF, SMF, UPF, NRF, AUSF, UDM, PCF |
| **UERANSIM** | Simulates gNB (base station) and UE (device) in software |
| **srsRAN** | Full 5G RAN stack — transmits real NR signals via USRP B210 |
| **MongoDB** | Database for subscriber profiles and session data |
| **WebUI** | Browser-based dashboard for subscriber management (port 9999) |
| **USRP B210** | Software-defined radio hardware for live RF transmission (Part 2) |

---

## 🏗️ Architecture Overview

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
│  ├── SMF                │◄──N2──►  └── nr-ue  (UE)        │
│  ├── UPF  (port 2152)   │◄──N3──►                         │
│  ├── NRF                │       │  OR  srsRAN:             │
│  ├── AUSF, UDM, PCF     │       │  └── gnb (real RF)      │
│  ├── MongoDB (27017)    │       │                         │
│  └── WebUI (port 9999)  │       │                         │
└─────────────────────────┘       └─────────────────────────┘

N2 = Control Plane (NGAP/SCTP, port 38412)
N3 = User Plane   (GTP-U/UDP,  port 2152)
```

---

## 💻 System Requirements

### Part 1 — Software Setup

| | VM1 — 5G Core | VM2 — RAN / UE |
|---|---|---|
| **OS** | Ubuntu 22.04 LTS | Ubuntu 22.04 LTS |
| **RAM** | 4 GB min / 8 GB recommended | 2 GB min / 4 GB recommended |
| **Storage** | 20 GB | 10 GB |
| **Network** | Must reach VM2 + internet | Must reach VM1 |

> Using VirtualBox or VMware? Set both adapters to **Bridged Mode**.

### Part 2 — Additional Hardware

| Item | Requirement |
|---|---|
| USRP B210 | USB **3.0** connection (USB 2.0 insufficient) |
| Programmable SIM | Blank SIM from Open-Cells or sysmocom |
| RAM | 16 GB recommended |
| CPU | Intel i7 or equivalent |

---

## ✅ Key Features

- ✅ Full 5G Core — AMF, SMF, UPF, NRF, AUSF, UDM, PCF
- ✅ Network Slicing — eMBB, URLLC, mMTC slice support
- ✅ RAN Simulation — virtual gNB + UE via UERANSIM
- ✅ MongoDB subscriber management
- ✅ WebUI for subscriber provisioning
- ✅ Real RF support via USRP B210 + srsRAN
- ✅ End-to-end validation and troubleshooting guide

---

---

# 🔧 Part 1: Open5GS + UERANSIM

---

## VM1 — 5G Core Setup

> All commands run on **VM1**

---

### Step 1 — Install MongoDB

```bash
sudo apt update
sudo apt install -y gnupg curl

curl -fsSL https://pgp.mongodb.com/server-8.0.asc | \
  sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg --dearmor

echo "deb [arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg] \
  https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/8.0 multiverse" | \
  sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list

sudo apt update
sudo apt install -y mongodb-org
sudo systemctl start mongod
sudo systemctl enable mongod
sudo systemctl status mongod
```

---

### Step 2 — Install Open5GS

```bash
sudo apt install -y software-properties-common
sudo add-apt-repository ppa:open5gs/latest
sudo apt update
sudo apt install -y open5gs

# Verify — should show 17 active services
sudo systemctl status open5gs-* | grep -c "active (running)"
```

---

### Step 3 — Install WebUI

```bash
sudo apt install -y ca-certificates curl gnupg
sudo mkdir -p /etc/apt/keyrings

curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg

NODE_MAJOR=20
echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] \
  https://deb.nodesource.com/node_$NODE_MAJOR.x nodistro main" | \
  sudo tee /etc/apt/sources.list.d/nodesource.list

sudo apt update && sudo apt install -y nodejs
curl -fsSL https://open5gs.org/open5gs/assets/webui/install | sudo -E bash -
```

Access at `http://<VM1-IP>:9999` or `http://localhost:9999` — default credentials: `admin` / `1423`

---

### Step 4 — Configure AMF

Open5GS binds to `127.x.x.x` by default. Update to VM1's network IP so the gNB on VM2 can reach it over the N2 interface.

```bash
sudo nano /etc/open5gs/amf.yaml
```

```yaml
amf:
  sbi:
    server:
      - address: <VM1-IP>       # was 127.0.0.5
  ngap:
    server:
      - address: <VM1-IP>       # N2 interface — gNB connects here
  metrics:
    server:
      - address: <VM1-IP>
        port: 9090
  guami:
    - plmn_id:
        mcc: 999
        mnc: 70
  tai:
    - plmn_id:
        mcc: 999
        mnc: 70
      tac: 1                    # must match gNB config
  plmn_support:
    - plmn_id:
        mcc: 999
        mnc: 70
      s_nssai:
        - sst: 1
```

```bash
sudo systemctl restart open5gs-amfd
sudo systemctl status open5gs-amfd
```

---

### Step 5 — Configure UPF

```bash
sudo nano /etc/open5gs/upf.yaml
```

```yaml
upf:
  gtpu:
    server:
      - address: <VM1-IP>       # N3 interface — was 127.0.0.7
  session:
    - subnet: 10.45.0.0/16
      gateway: 10.45.0.1
      dnn: internet
```

```bash
sudo systemctl restart open5gs-upfd
sudo systemctl status open5gs-upfd
```

---

### Step 6 — IP Forwarding & NAT

Required for UE traffic to reach the internet through the UPF.

```bash
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf

sudo iptables -t nat -A POSTROUTING -o $(ip route show default | awk '/default/ {print $5}') -j MASQUERADE
sudo iptables -I FORWARD 1 -j ACCEPT

sudo systemctl stop ufw && sudo systemctl disable ufw

sudo apt install -y iptables-persistent
sudo dpkg-reconfigure iptables-persistent

# Verify
cat /proc/sys/net/ipv4/ip_forward   # must output: 1
```

---

### Step 7 — Add Subscriber (WebUI)

Go to `http://<VM1-IP>:9999` → **Subscribers** → **+**

| Field | Value |
|---|---|
| IMSI | `999700000000001` |
| Subscriber Key (K) | `465B5CE8B199B49FAA5F0A2EE238A6BC` |
| Operator Key Type | `OPc` |
| OPc | `E8ED289DEBA952E4283B54E88E6183CA` |
| DNN | `internet` |

> ⚠️ These values must match exactly with the UE config in Step 10.

---

## VM2 — RAN & UE Setup (UERANSIM)

> All commands run on **VM2**

---

### Step 8 — Build UERANSIM

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y make g++ libsctp-dev lksctp-tools iproute2
sudo snap install cmake --classic

git clone https://github.com/aligungr/UERANSIM
cd UERANSIM
make
```

Produces: `build/nr-gnb` (gNB) and `build/nr-ue` (UE)

---

### Step 9 — Configure gNB

```bash
nano config/open5gs-gnb.yaml
```

```yaml
mcc: '999'
mnc: '70'
nci: '0x000000010'
idLength: 32
tac: 1                   # must match AMF tac

linkIp: <VM2-IP>
ngapIp: <VM2-IP>
gtpIp: <VM2-IP>

amfConfigs:
  - address: <VM1-IP>    # AMF on VM1
    port: 38412

slices:
  - sst: 1

ignoreStreamIds: true
```

---

### Step 10 — Configure UE

```bash
nano config/open5gs-ue.yaml
```

```yaml
supi: 'imsi-999700000000001'
mcc: '999'
mnc: '70'

key: '465B5CE8B199B49FAA5F0A2EE238A6BC'   # must match WebUI subscriber
op: 'E8ED289DEBA952E4283B54E88E6183CA'    # OPc value
opType: 'OPC'

gnbSearchList:
  - <VM2-IP>

sessions:
  - type: 'IPv4'
    apn: 'internet'
    slice:
      sst: 1
```

---

## 🚀 Running the Network

### Step 11 — Verify Core (VM1)

```bash
sudo systemctl status open5gs-*
# All services: active (running)
```

### Step 12 — Start gNB (VM2)

```bash
cd ~/UERANSIM
./build/nr-gnb -c config/open5gs-gnb.yaml
```

Expected:
```
[INFO] NG Setup procedure is successful  ✅
```

> Leave this terminal open.

### Step 13 — Start UE (VM2, new terminal)

```bash
cd ~/UERANSIM
sudo ./build/nr-ue -c config/open5gs-ue.yaml
```

Expected:
```
[INFO] UE switches to state: MM-REGISTERED/NORMAL-SERVICE
[INFO] PDU Session Establishment successful
[INFO] TUN interface[uesimtun0, 10.45.0.x] is up  ✅
```

---

## ✅ Testing

```bash
# Confirm TUN interface
ip addr show uesimtun0

# Ping through 5G core
ping -I uesimtun0 8.8.8.8

# HTTP request via 5G interface
curl --interface uesimtun0 https://httpbin.org/get

# Monitor core logs (VM1)
sudo tail -f /var/log/open5gs/amf.log
sudo tail -f /var/log/open5gs/upf.log
```

---

---

# 🔌 Part 2: Open5GS + srsRAN + USRP B210

> The 5G Core (VM1) setup is identical to Part 1. Only the RAN side changes.

---

### Step 1 — Install srsRAN Dependencies

```bash
sudo apt update
sudo apt install -y cmake make gcc g++ pkg-config git \
  libfftw3-dev libmbedtls-dev libsctp-dev \
  libyaml-cpp-dev libgtest-dev libuhd-dev uhd-host
```

---

### Step 2 — USRP B210 Setup

```bash
sudo python3 /usr/lib/uhd/utils/uhd_images_downloader.py
```

Connect USRP B210 via **USB 3.0**, then verify:

```bash
uhd_find_devices     # should show: product: B210
uhd_usrp_probe       # full hardware info — no errors = working ✅
```

---

### Step 3 — Build srsRAN

```bash
git clone https://github.com/srsRAN/srsRAN_Project.git
cd srsRAN_Project
mkdir build && cd build
cmake ../
make -j $(nproc)
sudo make install
```

---

### Step 4 — Configure srsRAN gNB

```bash
cd srsRAN_Project/build/apps/gnb/
cp gnb_rf_b200_tdd_n78_20mhz.yml my_gnb.yml
sudo nano my_gnb.yml
```

```yaml
cu_cp:
  amf:
    addr: <VM1-IP>
    port: 38412
    bind_addr: <THIS-IP>
    supported_tracking_areas:
      - tac: 7                  # update AMF tac to 7 as well
        plmn_list:
          - plmn: "99970"       # MCC999 + MNC70

cell_cfg:
  dl_arfcn: 632628
  band: 78
  channel_bandwidth_MHz: 20
  common_scs: 30
  plmn: "99970"
  tac: 7
  pci: 1
```

> ⚠️ Update `tac` in `/etc/open5gs/amf.yaml` on VM1 to `7` to match.

---

### Step 5 — Program SIM Card

```bash
tar -xvzf uicc-v3.3.tgz && cd uicc-v3.3 && make

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

---

### Step 6 — Register Hardware Subscriber (WebUI)

Go to `http://localhost:9999` or `http://<VM1-IP>:9999` → **Subscribers** → **+**

| Field | Value |
|---|---|
| IMSI | `999700000000001` |
| Key (K) | `6874736969202073796d4b2079650a73` |
| Operator Key Type | `OPc` |
| OPc | `504f20634f6320504f50206363500a4f` |
| DNN | `internet` |

> ⚠️ These keys differ from Part 1 — use the values above, matching what was programmed onto the SIM.

---

### Step 7 — Run the Network

```bash
# VM1 — confirm core is up
sudo systemctl status open5gs-*

# Start srsRAN gNB
cd srsRAN_Project/build/apps/gnb/
sudo ./gnb -c my_gnb.yml
```

Expected:
```
[INFO] B210 found
[INFO] Cell pci=1, bw=20 MHz
[INFO] Connected to AMF  ✅
```

Power on the UE device with the programmed SIM. It will register to **"PrivNet"** automatically.

---

---

## 🔧 Troubleshooting

### MongoDB won't start
```bash
sudo journalctl -u mongod -n 50
sudo chown -R mongodb:mongodb /var/lib/mongodb
sudo systemctl restart mongod
```

### Open5GS service failed
```bash
sudo systemctl status open5gs-amfd
sudo journalctl -u open5gs-amfd -n 30
sudo systemctl restart open5gs-*
```

### gNB can't connect to AMF — NG Setup fails

- [ ] Is `<VM1-IP>` correct in gNB config under `amfConfigs.address`?
- [ ] Is AMF listening on VM1's real IP (not `127.0.0.5`) in `/etc/open5gs/amf.yaml`?
- [ ] Can VM2 reach VM1? → `ping <VM1-IP>`
- [ ] Is port 38412 reachable? → `nc -zv <VM1-IP> 38412`
- [ ] Does `tac` match in both gNB and AMF configs?
- [ ] Do `mcc` / `mnc` match in both configs?

### UE authentication fails

Compare **character by character** between `config/open5gs-ue.yaml` and the WebUI subscriber:
- `supi` (without `imsi-` prefix) = IMSI
- `key` = Subscriber Key K
- `op` = OPc value

### UE registered but no internet

```bash
cat /proc/sys/net/ipv4/ip_forward          # must be: 1
sudo iptables -t nat -L POSTROUTING -n -v  # must show MASQUERADE rule
sudo ufw status                            # must be: inactive
```

### USRP B210 not detected

```bash
lsusb | grep Ettus   # verify OS sees the device
# Try a different USB 3.0 port — USB 2.0 is insufficient
sudo apt install --reinstall libuhd-dev uhd-host
sudo python3 /usr/lib/uhd/utils/uhd_images_downloader.py
```

---

## ❓ FAQ

**Q: Can both VMs run on one physical machine?**  
Yes — use VirtualBox/VMware with both adapters in **Bridged Mode**. Minimum 8 GB RAM total.

**Q: Single machine setup?**  
Open5GS and UERANSIM support loopback (`127.x.x.x`) single-host mode. See the UERANSIM GitHub wiki.

**Q: Ping works but DNS doesn't?**  
```bash
resolvectl dns uesimtun0 8.8.8.8
```

**Q: What's the difference between K and OPc?**  
`K` is the master SIM secret key. `OPc` is derived from `K` + operator key via a one-way function — safer to store on the network side. Both must match identically between UE/SIM and the core database.

**Q: How many UEs can UERANSIM simulate?**  
Multiple — launch `nr-ue` multiple times with separate config files and unique IMSI values. Each needs its own WebUI subscriber entry. Limit is CPU/RAM.

**Q: Is this production-ready?**  
No — this is a lab/research setup. For any real deployment: rotate all default credentials, configure strict firewall rules, restrict WebUI to trusted IPs, and schedule MongoDB backups.

---

## 📚 References

- [Open5GS Documentation](https://open5gs.org/open5gs/docs/)
- [UERANSIM GitHub](https://github.com/aligungr/UERANSIM)
- [srsRAN Project Docs](https://docs.srsran.com/projects/project)
- [Open-Cells UICC Programming Tool](https://open-cells.com/index.php/uiccsim-programing/)
- [Ettus UHD Documentation](https://files.ettus.com/manual/)
- [3GPP TS 23.501 — 5G System Architecture](https://www.3gpp.org/ftp/Specs/archive/23_series/23.501/)

---

## ⚖️ Disclaimer

This project is for **educational and research purposes only**. All testing must be conducted in a controlled lab environment. Before deploying any radio-transmitting equipment, verify compliance with local RF regulations and obtain required licenses. The PLMN codes used (MCC=999, MNC=70) are reserved for test networks and are not valid on public infrastructure.

---

> 💡 Found this useful? Star the repo and share it with others in the 5G space.
