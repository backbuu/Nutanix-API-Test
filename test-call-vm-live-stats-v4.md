# Nutanix API v4 — VM Live Stats (v4 Only)

Live performance metrics for VMs using **only the v4 VMM stats endpoint** — no v1 REST fallback.

Verified on: **PC `pc.7.3.1.3` / AOS `7.3` / AHV `10.3`**

---

## Why v4 Stats?

Before PC `pc.2024.1` / AOS `6.8`, VM live stats were only available via the deprecated Prism Element v1 REST API (`/PrismGateway/services/rest/v1/vms/{uuid}`). The v4 stats endpoint (`/api/vmm/v4.0.b1/ahv/stats/vms/{extId}`) is the official replacement — it returns time-series data with selectable metrics, configurable sampling intervals, and OData-compatible filtering.

---

## Endpoint

```
GET https://{pc_ip}:9440/api/vmm/v4.0.b1/ahv/stats/vms/{vmExtId}
```

### Query Parameters

| Parameter | Required | Format | Description |
|---|---|---|---|
| `$select` | ✅ Yes | `stats/fieldName,...` | Comma-separated metric fields — must be prefixed with `stats/` |
| `$startTime` | ✅ Yes | ISO-8601 URL-encoded | e.g. `2026-06-25T01:00:00.000000%2B00:00` |
| `$endTime` | ✅ Yes | ISO-8601 URL-encoded | e.g. `2026-06-25T02:00:00.000000%2B00:00` |
| `$samplingInterval` | ✅ Yes | int (seconds) | `30` = 30-second buckets. Min `30`, common values: `30`, `60`, `300` |
| `$statType` | ✅ Yes | enum | `LAST`, `MIN`, `MAX`, `SUM`, `AVG`, `COUNT` |

> **All four parameters are required.** Omitting any one returns error `VMM-30102`.
> The `+` in the timezone offset must be URL-encoded as `%2B`.

---

## Available Metric Fields (`$select`)

All fields must be prefixed with `stats/` in the `$select` parameter.

### CPU

| Field | Unit | Description |
|---|---|---|
| `stats/hypervisorCpuUsagePpm` | ppm | CPU usage — divide by `10,000` for % |
| `stats/hypervisorCpuReadyTimePpm` | ppm | CPU ready time (vCPU waiting for physical CPU) |

### Memory

| Field | Unit | Description |
|---|---|---|
| `stats/memoryUsagePpm` | ppm | Host-reported memory usage |
| `stats/guestMemoryUsagePpm` | ppm | Guest OS-reported memory usage (requires NGT) |

### Disk I/O

| Field | Unit | Description |
|---|---|---|
| `stats/controllerNumIops` | count | Total IOPS (read + write) |
| `stats/controllerIoBandwidthKbps` | kBps | Total I/O bandwidth |
| `stats/controllerReadIoBandwidthKbps` | kBps | Read bandwidth |
| `stats/controllerWriteIoBandwidthKbps` | kBps | Write bandwidth |
| `stats/controllerAvgIoLatencyMicros` | µs | Average I/O latency |
| `stats/controllerNumReadIo` | count | Read I/O operations |
| `stats/controllerNumWriteIo` | count | Write I/O operations |

### Network

| Field | Unit | Description |
|---|---|---|
| `stats/hypervisorNumReceivedBytes` | bytes | Bytes received per interval |
| `stats/hypervisorNumTransmittedBytes` | bytes | Bytes transmitted per interval |
| `stats/hypervisorNumReceivePacketsDropped` | count | Dropped inbound packets |
| `stats/hypervisorNumTransmitPacketsDropped` | count | Dropped outbound packets |

---

## `$statType` Values

| Value | Description |
|---|---|
| `LAST` | Last sample in the interval — best for "current" snapshot |
| `MIN` | Minimum value across the interval |
| `MAX` | Peak value across the interval |
| `AVG` | Average value across the interval |
| `SUM` | Sum of all samples — useful for counters |
| `COUNT` | Number of samples collected |

---

## cURL Example

```bash
PC="{pc_ip}"
AUTH="{username}:{password}"
VM_ID="{vmExtId}"

# Build time range (last 15 minutes, UTC)
START=$(python3 -c "
from urllib.parse import quote
from datetime import datetime, timezone, timedelta
t = datetime.now(timezone.utc) - timedelta(minutes=15)
print(quote(t.strftime('%Y-%m-%dT%H:%M:%S.000000+00:00')))
")
END=$(python3 -c "
from urllib.parse import quote
from datetime import datetime, timezone
print(quote(datetime.now(timezone.utc).strftime('%Y-%m-%dT%H:%M:%S.000000+00:00')))
")

SELECT="stats/hypervisorCpuUsagePpm,\
stats/hypervisorCpuReadyTimePpm,\
stats/memoryUsagePpm,\
stats/guestMemoryUsagePpm,\
stats/controllerNumIops,\
stats/controllerIoBandwidthKbps,\
stats/controllerAvgIoLatencyMicros,\
stats/controllerReadIoBandwidthKbps,\
stats/controllerWriteIoBandwidthKbps,\
stats/hypervisorNumReceivedBytes,\
stats/hypervisorNumTransmittedBytes,\
stats/hypervisorNumReceivePacketsDropped,\
stats/hypervisorNumTransmitPacketsDropped"

curl -sk -u "$AUTH" \
  "https://$PC:9440/api/vmm/v4.0.b1/ahv/stats/vms/$VM_ID?\$select=$SELECT&\$startTime=$START&\$endTime=$END&\$samplingInterval=30&\$statType=LAST" \
  | python3 -m json.tool
```

### Sample Response

```json
{
  "data": {
    "extId": "c90699b2-ec53-4800-93e7-4cf23024b75e",
    "stats": [
      {
        "hypervisorCpuUsagePpm": 3343,
        "hypervisorCpuReadyTimePpm": 2,
        "memoryUsagePpm": 399315,
        "guestMemoryUsagePpm": 419255,
        "controllerNumIops": 1,
        "controllerIoBandwidthKbps": 12,
        "controllerAvgIoLatencyMicros": 367,
        "controllerReadIoBandwidthKbps": 5,
        "controllerWriteIoBandwidthKbps": 7,
        "hypervisorNumReceivedBytes": 18387,
        "hypervisorNumTransmittedBytes": 21002,
        "hypervisorNumReceivePacketsDropped": 0,
        "hypervisorNumTransmitPacketsDropped": 0
      }
    ]
  }
}
```

---

## Full Python Script (v4 Only)

Fetches all VMs via v4 config, pulls live stats via v4 stats endpoint, prints a summary table.

```bash
pip install ntnx-vmm-py-client urllib3 tabulate
```

```python
"""
Nutanix VM Live Stats — v4 API only
Sources: vmm/v4.0.b1/ahv/config/vms (config) + vmm/v4.0.b1/ahv/stats/vms (live stats)
Verified on: PC pc.7.3.1.3 / AOS 7.3 / AHV 10.3
"""

import time
import urllib.parse
from datetime import datetime, timezone, timedelta

import urllib3
import ntnx_vmm_py_client as vmm
from ntnx_vmm_py_client.models.common.v1.stats import DownSamplingOperator
from tabulate import tabulate

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

PC_IP     = "{pc_ip}"
PORT      = 9440
USERNAME  = "{username}"
PASSWORD  = "{password}"

# Stats window — last 15 minutes, 30-second buckets
WINDOW_MINUTES  = 15
SAMPLING_SECS   = 30
STAT_TYPE       = DownSamplingOperator.DownSamplingOperator.LAST

# All metrics to fetch
SELECT_FIELDS = [
    "stats/hypervisorCpuUsagePpm",
    "stats/hypervisorCpuReadyTimePpm",
    "stats/memoryUsagePpm",
    "stats/guestMemoryUsagePpm",
    "stats/controllerNumIops",
    "stats/controllerIoBandwidthKbps",
    "stats/controllerAvgIoLatencyMicros",
    "stats/controllerReadIoBandwidthKbps",
    "stats/controllerWriteIoBandwidthKbps",
    "stats/hypervisorNumReceivedBytes",
    "stats/hypervisorNumTransmittedBytes",
    "stats/hypervisorNumReceivePacketsDropped",
    "stats/hypervisorNumTransmitPacketsDropped",
]


def make_client():
    config = vmm.Configuration()
    config.host       = PC_IP
    config.port       = PORT
    config.username   = USERNAME
    config.password   = PASSWORD
    config.verify_ssl = False
    client = vmm.ApiClient(configuration=config)
    client.add_default_header("Accept-Encoding", "gzip, deflate, br")
    return client


def list_all_vms(client):
    api  = vmm.VmsApi(api_client=client)
    page = 0
    vms  = []
    while True:
        resp  = api.list_vms(_page=page, _limit=100)
        batch = resp.data or []
        if not batch:
            break
        vms.extend(batch)
        page += 1
    return vms


def get_vm_stats(client, vm_ext_id):
    api = vmm.StatsApi(api_client=client)
    now   = datetime.now(timezone.utc)
    start = now - timedelta(minutes=WINDOW_MINUTES)

    resp = api.get_vm_stats_by_id(
        ext_id          = vm_ext_id,
        _select         = ",".join(SELECT_FIELDS),
        _start_time     = start,
        _end_time       = now,
        _sampling_interval = SAMPLING_SECS,
        _stat_type      = STAT_TYPE,
        async_req       = False,
    )
    tuples = resp.data.stats if resp.data and resp.data.stats else []
    return tuples[-1] if tuples else None  # most recent sample


def fmt_ppm(ppm):
    if ppm is None or ppm < 0:
        return "N/A"
    return f"{ppm / 10000:.1f}%"

def fmt_kbps(kbps):
    if kbps is None or kbps < 0:
        return "N/A"
    if kbps >= 1024:
        return f"{kbps / 1024:.1f} MBps"
    return f"{kbps} kBps"

def fmt_bytes(b):
    if b is None or b < 0:
        return "N/A"
    if b >= 1024 * 1024:
        return f"{b / 1024**2:.1f} MB"
    if b >= 1024:
        return f"{b / 1024:.1f} KB"
    return f"{b} B"

def fmt_latency(us):
    if us is None or us < 0:
        return "N/A"
    if us >= 1000:
        return f"{us / 1000:.1f} ms"
    return f"{us} µs"

def get_ips(vm):
    ips = []
    for nic in (vm.nics or []):
        nni = nic.nic_network_info or nic.network_info
        if nni and nni.ipv4_config and nni.ipv4_config.ip_address:
            ips.append(nni.ipv4_config.ip_address.value)
    return ", ".join(ips) or "—"


def main():
    client = make_client()

    print("Fetching VM list...")
    vms = list_all_vms(client)
    print(f"Found {len(vms)} VMs. Fetching live stats...\n")

    rows = []
    for vm in vms:
        power = vm.power_state.value if vm.power_state else "—"
        vcpus = (vm.num_sockets or 0) * (vm.num_cores_per_socket or 1)
        mem_gb = f"{(vm.memory_size_bytes or 0) / 1024**3:.1f} GB"
        ips   = get_ips(vm)

        s = get_vm_stats(client, vm.ext_id) if power == "ON" else None

        rows.append([
            vm.name,
            power,
            ips,
            vcpus,
            mem_gb,
            fmt_ppm(s.hypervisor_cpu_usage_ppm if s else None),
            fmt_ppm(s.hypervisor_cpu_ready_time_ppm if s else None),
            fmt_ppm(s.guest_memory_usage_ppm if s else None),
            f"{s.controller_num_iops}" if s else "—",
            fmt_kbps(s.controller_io_bandwidth_kbps if s else None),
            fmt_latency(s.controller_avg_io_latency_micros if s else None),
            fmt_bytes(s.hypervisor_num_received_bytes if s else None),
            fmt_bytes(s.hypervisor_num_transmitted_bytes if s else None),
        ])

    headers = [
        "Name", "Power", "IP", "vCPU", "RAM",
        "CPU%", "CPU Ready",
        "Mem% (guest)",
        "IOPS", "Disk BW", "Disk Lat",
        "Net RX", "Net TX",
    ]
    print(tabulate(rows, headers=headers, tablefmt="github"))


if __name__ == "__main__":
    main()
```

---

## Key Rules for v4 Stats

| Rule | Detail |
|---|---|
| `$select` prefix | Every field **must** start with `stats/` — e.g. `stats/hypervisorCpuUsagePpm` |
| Time format | ISO-8601, URL-encode `+` as `%2B` — e.g. `2026-06-25T01:00:00.000000%2B00:00` |
| All 4 params required | `$select`, `$startTime`, `$endTime`, `$samplingInterval` are all mandatory |
| Stats are time-series | Response returns **one tuple per sampling interval** in the window — use the last element for "current" value |
| OFF VMs return no stats | Skip stats calls for VMs where `powerState != ON` |
| `guestMemoryUsagePpm` | Requires Nutanix Guest Tools (NGT) to be installed inside the VM |
| Minimum interval | `$samplingInterval=30` is the minimum — data is collected every 30 seconds |

---

## Metric Conversion Reference

| Metric | Raw Value | Formula | Result |
|---|---|---|---|
| CPU usage | `3343 ppm` | `÷ 10,000` | `0.33%` |
| CPU ready | `2 ppm` | `÷ 10,000` | `0.0002%` |
| Memory (guest) | `419255 ppm` | `÷ 10,000` | `41.9%` |
| Disk latency | `367 µs` | `÷ 1,000` | `0.37 ms` |
| Disk bandwidth | `12 kBps` | `× 1` | `12 kBps` |
| Net received | `18387 bytes` | `÷ 1,024` | `17.9 KB` per 30s |

---

## Live Test Results (VM: `autoad` — 30s sample)

| Metric | Value | Converted |
|---|---|---|
| CPU usage | `3343 ppm` | `0.33%` |
| CPU ready time | `2 ppm` | `~0%` |
| Memory usage (host) | `399315 ppm` | `39.9%` |
| Memory usage (guest) | `419255 ppm` | `41.9%` |
| Disk IOPS | `1` | 1 IOPS |
| Disk bandwidth (total) | `12 kBps` | 12 kBps |
| Disk read bandwidth | `5 kBps` | 5 kBps |
| Disk write bandwidth | `7 kBps` | 7 kBps |
| Disk latency (avg) | `367 µs` | 0.37 ms |
| Network received | `18,387 bytes` | 17.9 KB / 30s |
| Network transmitted | `21,002 bytes` | 20.5 KB / 30s |
| Packets dropped (RX) | `0` | — |
| Packets dropped (TX) | `0` | — |

---

## References

- [Nutanix VMM API Reference — Stats](https://developers.nutanix.com/api-reference?namespace=vmm&version=v4.0.b1)
- [VmStats Python SDK Model](https://developers.nutanix.com/api/v1/sdk/namespaces/main/vmm/versions/v4.0/languages/python/ntnx_vmm_py_client.models.vmm.v4.ahv.stats.VmStats.html)
- [New v4 API features in PC 2024.1 / AOS 6.8](https://www.nutanix.dev/2024/06/18/new-v4-api-and-sdk-features-in-prism-central-2024-1-and-aos-6-8/)
- [VmStatsTuple Go struct (field reference)](https://developers.golang.nutanix.com/github.com/nutanix/ntnx-api-golang-clients/vmm-go-client/v4/models/vmm/v4/ahv/stats)
