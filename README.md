# Nutanix API v4 — VM Info (CPU, Memory, Disk)

Reference and test scripts for querying VM resource data via the Nutanix VMM v4 API.

---

## Prerequisites

- Prism Central pc.2024.3+ / AOS 7.0+
- Python 3.8+

```bash
pip install ntnx-vmm-py-client
```

---

## Authentication

| Method | Header |
|---|---|
| Basic Auth | `Authorization: Basic <base64(user:pass)>` |
| API Key (recommended for automation) | `X-Ntnx-Api-Key: <key>` |

---

## Endpoint

```
GET https://{prism_central_ip}:9440/api/vmm/v4.1/ahv/config/vms
```

### Query Parameters

| Parameter | Type | Default | Notes |
|---|---|---|---|
| `$page` | int | 0 | Zero-based page number |
| `$limit` | int | 50 | Max 100 per request |
| `$filter` | string | — | OData v4.01 filter expression |
| `$select` | string | — | Comma-separated fields to return |

### Example cURL

```bash
curl -sk -u admin:password \
  "https://10.0.0.1:9440/api/vmm/v4.1/ahv/config/vms?$limit=100&$select=extId,name,numSockets,numCoresPerSocket,memorySizeBytes,disks" \
  | python3 -m json.tool
```

---

## Key Response Fields

| Field | Type | Description |
|---|---|---|
| `extId` | string | VM UUID |
| `name` | string | VM display name |
| `numSockets` | int | Number of vCPU sockets |
| `numCoresPerSocket` | int | Cores per socket |
| `memorySizeBytes` | long | RAM in bytes |
| `disks[].diskSizeBytes` | long | Disk capacity in bytes |
| `disks[].diskAddress.busType` | string | `SCSI`, `SATA`, or `IDE` |

**Derived values:**

```
Total vCPU  = numSockets × numCoresPerSocket
RAM (GiB)   = memorySizeBytes / (1024³)
Disk (GiB)  = diskSizeBytes / (1024³)
```

---

## Python SDK — List All VMs

```python
import ntnx_vmm_py_client as vmm

# Configure connection
config = vmm.Configuration()
config.host = "https://10.0.0.1:9440"
config.username = "admin"
config.password = "password"
config.verify_ssl = False  # set True in production with valid cert

client = vmm.ApiClient(configuration=config)
api = vmm.VmsApi(api_client=client)

# Paginate through all VMs
page = 0
all_vms = []
while True:
    resp = api.list_vms(_page=page, _limit=100)
    vms = resp.data
    if not vms:
        break
    all_vms.extend(vms)
    page += 1

# Print summary
print(f"{'VM Name':<40} {'vCPU':>6} {'RAM (GiB)':>10} {'Disk (GiB)':>12}")
print("-" * 72)
for vm in all_vms:
    vcpu = (vm.num_sockets or 0) * (vm.num_cores_per_socket or 0)
    ram_gib = (vm.memory_size_bytes or 0) / (1024 ** 3)
    disk_gib = sum(
        (d.disk_size_bytes or 0) for d in (vm.disks or [])
    ) / (1024 ** 3)
    print(f"{vm.name:<40} {vcpu:>6} {ram_gib:>10.1f} {disk_gib:>12.1f}")
```

---

## Pagination Note

The v4 API caps responses at **100 VMs per request**. Always loop with `$page` until the response `data` array is empty — unlike v2/v3 which returned all VMs in a single call.

---

## References

- [Nutanix v4 API User Guide](https://www.nutanix.dev/nutanix-api-user-guide/)
- [VMM API Reference](https://developers.nutanix.com/api-reference?namespace=vmm&version=v4.0.b1)
- [Python SDK Docs — VmsApi](https://developers.nutanix.com/api/v1/sdk/namespaces/main/vmm/versions/v4.0/languages/python/ntnx_vmm_py_client.api.vm_api.html)
- [API Versions](https://www.nutanix.dev/api-versions/)
