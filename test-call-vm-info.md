# Nutanix API — Test: VM Information (Name, Network, IP, CPU, Memory, Disk, Usage, Power State)

Tested against Prism Central 9440. Two APIs are used together:
- **v4 VMM** (`vmm/v4.0.b1/ahv/config/vms`) — static config: name, CPU, memory, disk size, NICs, power state
- **v1 REST** (`PrismGateway/services/rest/v1/vms`) — live stats: CPU %, memory %, disk usage, IOPS, network bytes

---

## Environment

| Variable | Placeholder |
|---|---|
| Prism Central IP | `{pc_ip}` |
| Port | `9440` |
| Username | `{username}` |
| Password | `{password}` |

---

## API Overview

| Data Category | API | Endpoint |
|---|---|---|
| Name, CPU, Memory, Disk size, NICs, Power state | VMM v4 | `GET /api/vmm/v4.0.b1/ahv/config/vms` |
| CPU %, Memory %, Disk IOPS, Network bytes (live) | v1 REST | `GET /PrismGateway/services/rest/v1/vms/{uuid}` |
| List all VMs (v3) | v3 REST | `POST /api/nutanix/v3/vms/list` |

---

## Step 1 — Probe VMM Version

```bash
for ver in "v4.0.b1" "v4.0.b2" "v4.0" "v4.1" "v4.2"; do
  code=$(curl -sk -u "{username}:{password}" \
    "https://{pc_ip}:9440/api/vmm/$ver/ahv/config/vms?\$limit=1" \
    -o /dev/null -w "%{http_code}")
  echo "vmm/$ver → HTTP $code"
done
```

**Result from this test:**

| Version | Status |
|---|---|
| `v4.0.b1` | ✅ 200 — use this |
| `v4.0.b2` | ❌ 404 |
| `v4.0` | ❌ 404 |
| `v4.1` | ❌ 404 |

> Note: the VMM VM path is `/ahv/config/vms` — not `/vms` or `/ahv/vms`.

---

## Step 2 — List All VMs (v4 Config)

Returns name, CPU, memory, disk list, NIC list, power state for every VM.

```bash
curl -sk -u "{username}:{password}" \
  "https://{pc_ip}:9440/api/vmm/v4.0.b1/ahv/config/vms?\$limit=100" \
  | python3 -m json.tool
```

### Key Response Fields

| Field | Description |
|---|---|
| `extId` | VM UUID — used for all per-VM calls |
| `name` | VM display name |
| `powerState` | `ON` / `OFF` / `PAUSED` |
| `numSockets` | CPU socket count |
| `numCoresPerSocket` | Cores per socket |
| `memorySizeBytes` | Total RAM in bytes |
| `disks[].backingInfo.diskSizeBytes` | Provisioned disk size in bytes |
| `disks[].diskAddress.busType` | `SCSI`, `IDE`, `SATA`, `PCI` |
| `nics[].nicNetworkInfo.subnet.extId` | Subnet UUID the NIC is on |
| `nics[].nicNetworkInfo.ipv4Config.ipAddress.value` | Assigned IP address |
| `nics[].nicBackingInfo.macAddress` | MAC address |
| `cluster.extId` | Cluster UUID the VM lives on |
| `host.extId` | Host UUID the VM is running on |

---

## Step 3 — Get Single VM (v4 Config)

```bash
curl -sk -u "{username}:{password}" \
  "https://{pc_ip}:9440/api/vmm/v4.0.b1/ahv/config/vms/{vmExtId}" \
  | python3 -m json.tool
```

---

## Step 4 — Get Live Stats (v1 REST)

The v4 stats endpoint requires a `$select` parameter but returns errors on this PC build. Use v1 instead — it returns all live metrics in a single call.

```bash
curl -sk -u "{username}:{password}" \
  "https://{pc_ip}:9440/PrismGateway/services/rest/v1/vms/{vmUuid}" \
  | python3 -m json.tool
```

### Key Stats Fields (inside `stats` object)

| Field | Unit | Description |
|---|---|---|
| `hypervisor_cpu_usage_ppm` | ppm | CPU usage — divide by 10000 for % |
| `memory_usage_ppm` | ppm | Memory usage (host view) — divide by 10000 for % |
| `guest.memory_usage_ppm` | ppm | Memory usage (guest view) — divide by 10000 for % |
| `controller_num_iops` | count | Total IOPS |
| `controller_num_read_iops` | count | Read IOPS |
| `controller_num_write_iops` | count | Write IOPS |
| `controller_io_bandwidth_kBps` | kBps | Total disk I/O bandwidth |
| `controller_read_io_bandwidth_kBps` | kBps | Read bandwidth |
| `controller_write_io_bandwidth_kBps` | kBps | Write bandwidth |
| `controller_avg_io_latency_usecs` | µs | Average I/O latency |
| `controller.storage_tier.ssd.usage_bytes` | bytes | Actual SSD disk usage |
| `controller_user_bytes` | bytes | User data bytes stored |
| `hypervisor_num_received_bytes` | bytes | Network received (last interval) |
| `hypervisor_num_transmitted_bytes` | bytes | Network transmitted (last interval) |

### Key Top-Level Fields

| Field | Description |
|---|---|
| `vmName` | VM name |
| `powerState` | `on` / `off` |
| `numVCpus` | Total vCPUs (sockets × cores) |
| `memoryCapacityInBytes` | Configured RAM |
| `diskCapacityInBytes` | Total provisioned disk |
| `numNetworkAdapters` | NIC count |
| `ipAddresses` | List of IPs |
| `hostName` | AHV host the VM runs on |

---

## Sample Output (from live test — VM: `autoad`)

| Metric | Raw Value | Converted |
|---|---|---|
| Power State | `on` | ON |
| vCPUs | `2` | 2 vCPU (2 sockets × 1 core) |
| Memory (configured) | `4294967296 bytes` | 4 GB |
| Memory usage (guest) | `419375 ppm` | ~41.9% |
| CPU usage | `3671 ppm` | ~0.37% |
| Disk (provisioned) | `53687091200 bytes` | ~50 GB |
| Disk usage (SSD) | `34871964672 bytes` | ~32.5 GB |
| Disk IOPS | `1` | 1 IOPS |
| Disk bandwidth | `12 kBps` | 12 kBps total |
| Network received | `16570 bytes` | last 30s interval |
| Network transmitted | `19656 bytes` | last 30s interval |
| IP Address | `10.8.23.6` | — |
| Host | `RNO-POC012-3` | — |

---

## cURL Cheat Sheet

```bash
PC="{pc_ip}"
AUTH="{username}:{password}"
VM_ID="{vmExtId}"

# List all VMs (v4)
curl -sk -u "$AUTH" \
  "https://$PC:9440/api/vmm/v4.0.b1/ahv/config/vms?\$limit=100" \
  | python3 -m json.tool

# Get single VM config (v4)
curl -sk -u "$AUTH" \
  "https://$PC:9440/api/vmm/v4.0.b1/ahv/config/vms/$VM_ID" \
  | python3 -m json.tool

# Get live stats (v1)
curl -sk -u "$AUTH" \
  "https://$PC:9440/PrismGateway/services/rest/v1/vms/$VM_ID" \
  | python3 -m json.tool

# List VMs via v3 (with filter)
curl -sk -u "$AUTH" \
  -X POST "https://$PC:9440/api/nutanix/v3/vms/list" \
  -H "Content-Type: application/json" \
  -d '{"kind": "vm", "length": 100, "filter": "power_state==on"}' \
  | python3 -m json.tool
```

---

## Full Python Script

Fetches all VMs, merges v4 config with v1 live stats, prints a summary table.

```bash
pip install ntnx-vmm-py-client requests urllib3 tabulate
```

```python
"""
Nutanix VM Info — v4 config + v1 live stats
Prints: Name, IP, CPU cores, Memory, Disk size, CPU%, Mem%,
        Disk IOPS, Disk usage, Net RX/TX, Power state
"""

import requests
import urllib3
import ntnx_vmm_py_client as vmm
from tabulate import tabulate

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

PC_IP    = "{pc_ip}"
PORT     = 9440
USERNAME = "{username}"
PASSWORD = "{password}"
BASE_URL = f"https://{PC_IP}:{PORT}"
AUTH     = (USERNAME, PASSWORD)


def bytes_to_gb(b):
    if b is None or b < 0:
        return "N/A"
    return f"{b / 1024**3:.1f} GB"

def bytes_to_mb(b):
    if b is None or b < 0:
        return "N/A"
    return f"{b / 1024**2:.0f} MB"

def ppm_to_pct(ppm):
    if ppm is None or ppm < 0:
        return "N/A"
    return f"{ppm / 10000:.1f}%"

def kbps_fmt(kbps):
    if kbps is None or kbps < 0:
        return "N/A"
    if kbps >= 1024:
        return f"{kbps/1024:.1f} MBps"
    return f"{kbps} kBps"


# ── Step 1: list all VMs via v4 SDK ─────────────────────────────────────────

def get_all_vms_v4():
    config = vmm.Configuration()
    config.host       = PC_IP
    config.port       = PORT
    config.username   = USERNAME
    config.password   = PASSWORD
    config.verify_ssl = False

    client  = vmm.ApiClient(configuration=config)
    api     = vmm.VmsApi(api_client=client)

    all_vms = []
    page = 0
    while True:
        resp = api.list_vms(_page=page, _limit=100)
        batch = resp.data or []
        if not batch:
            break
        all_vms.extend(batch)
        page += 1
    return all_vms


# ── Step 2: get live stats via v1 REST ──────────────────────────────────────

def get_vm_stats_v1(vm_uuid):
    url = f"{BASE_URL}/PrismGateway/services/rest/v1/vms/{vm_uuid}"
    r = requests.get(url, auth=AUTH, verify=False)
    if r.status_code != 200:
        return {}, {}
    d = r.json()
    return d.get("stats", {}), d


# ── Step 3: merge and print ──────────────────────────────────────────────────

def build_report():
    print("Fetching VM list (v4)...")
    vms = get_all_vms_v4()
    print(f"Found {len(vms)} VMs. Fetching live stats...\n")

    rows = []
    for vm in vms:
        uuid  = vm.ext_id
        name  = vm.name or "—"
        power = vm.power_state.value if vm.power_state else "—"

        # CPU
        total_vcpus = (vm.num_sockets or 0) * (vm.num_cores_per_socket or 1)

        # Memory
        mem_bytes = vm.memory_size_bytes or 0

        # Disk — sum all disk sizes
        disk_bytes = 0
        if vm.disks:
            for d in vm.disks:
                bi = d.backing_info
                if bi and hasattr(bi, "disk_size_bytes") and bi.disk_size_bytes:
                    disk_bytes += bi.disk_size_bytes

        # NICs — collect IPs
        ips = []
        nic_names = []
        if vm.nics:
            for nic in vm.nics:
                nni = nic.nic_network_info or nic.network_info
                if nni:
                    subnet = nni.subnet.ext_id if nni.subnet else "—"
                    nic_names.append(subnet[:8] + "...")
                    ipcfg = nni.ipv4_config
                    if ipcfg and ipcfg.ip_address:
                        ips.append(ipcfg.ip_address.value)

        # Live stats from v1
        stats, v1 = get_vm_stats_v1(uuid)

        cpu_ppm     = stats.get("hypervisor_cpu_usage_ppm", -1)
        mem_ppm     = stats.get("guest.memory_usage_ppm", -1)
        iops        = stats.get("controller_num_iops", -1)
        bw_kbps     = stats.get("controller_io_bandwidth_kBps", -1)
        disk_used   = stats.get("controller.storage_tier.ssd.usage_bytes", -1)
        net_rx      = stats.get("hypervisor_num_received_bytes", -1)
        net_tx      = stats.get("hypervisor_num_transmitted_bytes", -1)
        host        = v1.get("hostName", "—")

        rows.append([
            name,
            power,
            ", ".join(ips) or "—",
            total_vcpus,
            bytes_to_gb(mem_bytes),
            ppm_to_pct(cpu_ppm),
            ppm_to_pct(mem_ppm),
            bytes_to_gb(disk_bytes),
            bytes_to_gb(disk_used),
            f"{iops}" if iops >= 0 else "—",
            kbps_fmt(bw_kbps),
            bytes_to_mb(net_rx) if net_rx >= 0 else "—",
            bytes_to_mb(net_tx) if net_tx >= 0 else "—",
            host,
        ])

    headers = [
        "Name", "Power", "IP", "vCPU", "RAM",
        "CPU%", "Mem%",
        "Disk Size", "Disk Used",
        "IOPS", "Disk BW",
        "Net RX", "Net TX",
        "Host"
    ]
    print(tabulate(rows, headers=headers, tablefmt="github"))


if __name__ == "__main__":
    build_report()
```

### Sample Output

```
| Name   | Power | IP          | vCPU | RAM   | CPU%  | Mem%  | Disk Size | Disk Used | IOPS | Disk BW | Net RX  | Net TX  | Host           |
|--------|-------|-------------|------|-------|-------|-------|-----------|-----------|------|---------|---------|---------|----------------|
| autoad | ON    | 10.x.x.x    | 2    | 4.0 GB| 0.4%  | 41.9% | 50.0 GB   | 32.5 GB   | 1    | 12 kBps | 0 MB    | 0 MB    | RNO-POC012-3   |
```

---

## Metric Reference

| Metric | Formula | Example |
|---|---|---|
| CPU % | `hypervisor_cpu_usage_ppm / 10000` | `3671 ppm → 0.37%` |
| Memory % | `guest.memory_usage_ppm / 10000` | `419375 ppm → 41.9%` |
| Disk size | `diskSizeBytes / 1024³` | `53687091200 → 50 GB` |
| Disk used | `controller.storage_tier.ssd.usage_bytes / 1024³` | `34871964672 → 32.5 GB` |
| Network | `hypervisor_num_received/transmitted_bytes` per 30s interval | cumulative bytes |

---

## Notes

- **`vmm/v4.0.b1/ahv/config/vms`** is the correct path — `/ahv/config/` is required, not `/ahv/` or `/config/`.
- **v4 stats endpoint** (`/ahv/stats/vms/{extId}`) requires a `$select` parameter that returned errors on this build. Use the v1 REST stats endpoint instead.
- `hypervisor_num_received_bytes` and `hypervisor_num_transmitted_bytes` are cumulative over the last polling interval (~30s), not per-second rates.
- `-1` values in stats mean the metric is not available for that VM at that moment (e.g. VM is off, or NGT not installed for guest memory).
- **Total vCPUs** = `numSockets × numCoresPerSocket`.
