# Nutanix API v4 — VPC & Projects

Tested on a live Prism Central environment. Covers version probing, VPC CRUD, Projects listing, and the VPC↔Project relationship.

---

## Test Environment

| Component | Version | Notes |
|-----------|---------|-------|
| Prism Central | `pc.7.3.1.3` | `el8.5-release-ganges-7.3.1.3` |
| AOS (RNO-POC012) | `7.3` | Primary PE cluster |
| AOS (DR) | `7.3` | DR PE cluster |
| PC Nodes | 1 | Single-node PC |
| Storage Type | All-Flash | |
| Architecture | X86_64 | |
| Timezone | America/Los_Angeles | |

### API Version Availability on This PC

| Namespace / Version | Status |
|--------------------|--------|
| `networking/v4.0.a1` | ❌ 404 |
| `networking/v4.0.b1` | ✅ 200 |
| `networking/v4.0.b2` | ✅ 200 |
| `networking/v4.0` | ✅ 200 |
| `networking/v4.1` | ✅ 200 |
| `networking/v4.1.a1` | ❌ 404 |
| `networking/v4.1.b1` | ❌ 404 |
| `networking/v4.2` | ❌ 404 |
| `iam/v4.0.b1` | ❌ 404 |
| `iam/v4.0.b2` | ❌ 404 |
| `iam/v4.0` | ❌ 404 |
| `iam/v4.1` | ❌ 404 |

**Conclusion:** Use `networking/v4.0` for VPCs. Use `v3` for Projects — IAM v4 projects endpoint is not available on PC 7.3.1.3.

---

## Namespace Overview

| Resource | Namespace | Endpoint Base | GA on this PC |
|----------|-----------|--------------|--------------|
| VPC | `networking` | `/api/networking/v4.0/config/vpcs` | ✅ Yes |
| Projects | `iam` (v4) | `/api/iam/v4.0/authz/projects` | ❌ Not available |
| Projects | v3 fallback | `/api/nutanix/v3/projects/list` | ✅ Yes |

---

## 1. Probe API Version (Always Do This First)

Different PC builds expose different API versions. Always probe before hardcoding a path.

```bash
for ver in "v4.0.b1" "v4.0.b2" "v4.0" "v4.1"; do
  code=$(curl -sk -u "{username}:{password}" \
    "https://{pc_ip}:9440/api/networking/$ver/config/vpcs?\$limit=1" \
    -o /dev/null -w "%{http_code}")
  echo "networking/$ver → HTTP $code"
done
```

---

## 2. VPC API

### List VPCs

```bash
curl -sk -u "{username}:{password}" \
  "https://{pc_ip}:9440/api/networking/v4.0/config/vpcs?\$limit=100" \
  | python3 -m json.tool
```

**Key response fields:**

| Field | Type | Description |
|-------|------|-------------|
| `data[].extId` | string | VPC UUID — used to link to projects |
| `data[].name` | string | VPC display name |
| `data[].vpcType` | string | `REGULAR` (NAT) or `TRANSIT` |
| `data[].externalSubnets[].subnetReference` | string | External subnet UUID (uplink) |
| `data[].externalSubnets[].externalIps` | array | SNAT IP pool |
| `data[].externalSubnets[].activeGatewayNodes` | array | HA gateway node IPs |
| `data[].commonDhcpOptions.domainNameServers` | array | DNS shared across overlay subnets |

**Live result — 1 VPC found:**

| Name | extId | Type | SNAT IPs | DNS | Gateway Nodes |
|------|-------|------|----------|-----|--------------|
| VPN-for-Test | `71f5d1e9-e7eb-4d2d-a3cb-d371d4f8310b` | REGULAR | `10.8.23.23`, `10.8.23.24` | `1.1.1.1` | `10.8.23.25`, `10.8.23.26` |

### Get VPC by extId

```bash
curl -sk -u "{username}:{password}" \
  "https://{pc_ip}:9440/api/networking/v4.0/config/vpcs/{vpcExtId}" \
  | python3 -m json.tool
```

> The `ETag` response header is required for PUT and DELETE calls — capture it from this GET.

### Create VPC

```bash
curl -sk -u "{username}:{password}" \
  -X POST "https://{pc_ip}:9440/api/networking/v4.0/config/vpcs" \
  -H "Content-Type: application/json" \
  -H "Ntnx-Request-Id: $(python3 -c 'import uuid; print(uuid.uuid4())')" \
  -d '{
    "name": "my-vpc",
    "description": "Test VPC"
  }' \
  | python3 -m json.tool
```

Returns a task `extId`. Poll it until `SUCCEEDED`:

```bash
curl -sk -u "{username}:{password}" \
  "https://{pc_ip}:9440/api/prism/v4.0.b1/config/tasks/{taskExtId}" \
  | python3 -m json.tool
```

### Update VPC

```bash
# Step 1: get ETag
ETAG=$(curl -sk -u "{username}:{password}" -I \
  "https://{pc_ip}:9440/api/networking/v4.0/config/vpcs/{vpcExtId}" \
  | grep -i 'etag' | awk '{print $2}' | tr -d '\r')

# Step 2: update
curl -sk -u "{username}:{password}" \
  -X PUT "https://{pc_ip}:9440/api/networking/v4.0/config/vpcs/{vpcExtId}" \
  -H "Content-Type: application/json" \
  -H "Ntnx-Request-Id: $(python3 -c 'import uuid; print(uuid.uuid4())')" \
  -H "If-Match: ${ETAG}" \
  -d '{"name": "my-vpc-updated", "description": "Updated"}' \
  | python3 -m json.tool
```

### Delete VPC

```bash
ETAG=$(curl -sk -u "{username}:{password}" -I \
  "https://{pc_ip}:9440/api/networking/v4.0/config/vpcs/{vpcExtId}" \
  | grep -i 'etag' | awk '{print $2}' | tr -d '\r')

curl -sk -u "{username}:{password}" \
  -X DELETE "https://{pc_ip}:9440/api/networking/v4.0/config/vpcs/{vpcExtId}" \
  -H "Ntnx-Request-Id: $(python3 -c 'import uuid; print(uuid.uuid4())')" \
  -H "If-Match: ${ETAG}" \
  | python3 -m json.tool
```

### Required Headers Summary

| Header | Required For | Value |
|--------|-------------|-------|
| `Content-Type` | POST, PUT | `application/json` |
| `Ntnx-Request-Id` | POST, PUT, DELETE | Fresh UUID v4 |
| `If-Match` | PUT, DELETE | ETag from prior GET |

---

## 3. Projects API

### v4 IAM — Not Available on PC 7.3.1.3

`GET /api/iam/v4.0/authz/projects` returns 404 on this environment. Use v3.

> v4 IAM projects requires a newer PC release. v3 is supported until Q4-CY2026.

### List Projects (v3)

```bash
curl -sk -u "{username}:{password}" \
  -X POST "https://{pc_ip}:9440/api/nutanix/v3/projects/list" \
  -H "Content-Type: application/json" \
  -d '{"kind": "project", "length": 100, "offset": 0}' \
  | python3 -m json.tool
```

**Key response fields:**

| Field | Description |
|-------|-------------|
| `entities[].metadata.uuid` | Project UUID |
| `entities[].status.name` | Project name |
| `entities[].status.state` | `COMPLETE` = healthy |
| `entities[].status.resources.vpc_reference_list` | VPCs attached to this project |
| `entities[].status.resources.subnet_reference_list` | Subnets available in project |
| `entities[].status.resources.default_subnet_reference` | Default subnet for new VMs |
| `entities[].status.resources.user_reference_list` | Users with access |
| `entities[].status.resources.external_user_group_reference_list` | AD/LDAP groups |
| `entities[].status.resources.cluster_reference_list` | Clusters scoped to project |

**Live result — 2 Projects found:**

| Project | UUID | VPC Linked | Default Subnet | Users / Groups |
|---------|------|-----------|----------------|---------------|
| Project-VPC | `e15d97b2-1b6b-43b3-829c-fa94f433a7a0` | VPN-for-Test ✅ | `secodary-for-vpc-49` | `adminuser02@ntnxlab.local` |
| NTNX | `e68b227a-8358-43cf-8697-8028e337b299` | none | `primary` | `CN=SSP Admins` (AD group) |

---

## 4. VPC ↔ Project Relationship

The `vpc_reference_list` in the v3 project response contains the VPC `extId` from the networking API — use it to join the two datasets.

```
VPN-for-Test (VPC)  extId: 71f5d1e9-e7eb-4d2d-a3cb-d371d4f8310b
  └── Project-VPC  uuid: e15d97b2-1b6b-43b3-829c-fa94f433a7a0
        ├── Subnets : overlay-49, secodary-for-vpc-49
        ├── Default : secodary-for-vpc-49 (external/uplink subnet)
        └── User    : adminuser02@ntnxlab.local

NTNX  uuid: e68b227a-8358-43cf-8697-8028e337b299
  └── (no VPC — flat VLAN subnets only)
        ├── Subnets : primary, secondary
        └── Group   : CN=SSP Admins,CN=Users,DC=ntnxlab,DC=local
```

---

## 5. Python Script — Full Workflow

Probes versions, lists VPCs, lists projects, prints the relationship map.

```bash
pip install ntnx-networking-py-client requests urllib3
```

```python
"""
Nutanix API v4 — VPC + Project relationship mapper
Tested on PC 7.3.1.3 / AOS 7.3
"""

import sys
import requests
import urllib3
import ntnx_networking_py_client as networking

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

PC_IP    = "{pc_ip}"
PORT     = 9440
USERNAME = "{username}"
PASSWORD = "{password}"
BASE_URL = f"https://{PC_IP}:{PORT}"
AUTH     = (USERNAME, PASSWORD)


def probe_networking_version():
    for ver in ["v4.0.b1", "v4.0.b2", "v4.0", "v4.1"]:
        url = f"{BASE_URL}/api/networking/{ver}/config/vpcs?$limit=1"
        r = requests.get(url, auth=AUTH, verify=False)
        print(f"networking/{ver} → HTTP {r.status_code}")
        if r.status_code == 200:
            print(f"  ✅ Using: networking/{ver}\n")
            return ver
    print("No supported networking version found.")
    sys.exit(1)


def list_vpcs(version):
    config = networking.Configuration()
    config.host = PC_IP
    config.port = PORT
    config.username = USERNAME
    config.password = PASSWORD
    config.verify_ssl = False

    client = networking.ApiClient(configuration=config)
    api = networking.VpcsApi(api_client=client)

    resp = api.list_vpcs(_page=0, _limit=100)
    vpcs = resp.data or []
    print(f"VPCs ({len(vpcs)} found):")
    for vpc in vpcs:
        snat_ips = []
        for es in (vpc.external_subnets or []):
            for ip in (es.external_ips or []):
                if ip.ipv4:
                    snat_ips.append(ip.ipv4.value)
        print(f"  • {vpc.name}")
        print(f"    extId : {vpc.ext_id}")
        print(f"    type  : {vpc.vpc_type}")
        print(f"    snat  : {snat_ips}")
    print()
    return vpcs


def list_projects_v3():
    url  = f"{BASE_URL}/api/nutanix/v3/projects/list"
    body = {"kind": "project", "length": 100, "offset": 0}
    r = requests.post(url, auth=AUTH, json=body, verify=False)
    r.raise_for_status()
    projects = r.json().get("entities", [])
    print(f"Projects ({len(projects)} found):")
    for p in projects:
        res     = p["status"]["resources"]
        name    = p["status"]["name"]
        uuid    = p["metadata"]["uuid"]
        vpc_ids = [v["uuid"] for v in res.get("vpc_reference_list", [])]
        subnets = [s["name"] for s in res.get("subnet_reference_list", [])]
        users   = [u["name"] for u in res.get("user_reference_list", [])]
        print(f"  • {name}  uuid={uuid}")
        print(f"    vpcs    : {vpc_ids or 'none'}")
        print(f"    subnets : {subnets}")
        print(f"    users   : {users or 'none'}")
    print()
    return projects


def print_relationship_map(vpcs, projects):
    vpc_lookup = {v.ext_id: v.name for v in vpcs}
    print("── VPC ↔ Project map ──────────────────────────")
    for p in projects:
        res      = p["status"]["resources"]
        name     = p["status"]["name"]
        vpc_refs = res.get("vpc_reference_list", [])
        subnets  = [s["name"] for s in res.get("subnet_reference_list", [])]
        if not vpc_refs:
            print(f"  {name}  →  (no VPC)  subnets={subnets}")
            continue
        for ref in vpc_refs:
            vpc_name = vpc_lookup.get(ref["uuid"], ref["uuid"])
            print(f"  {vpc_name}  →  {name}  subnets={subnets}")


if __name__ == "__main__":
    print("=== Step 1: Probe networking API version ===")
    net_ver = probe_networking_version()

    print("=== Step 2: List VPCs ===")
    vpcs = list_vpcs(net_ver)

    print("=== Step 3: List Projects (v3 fallback) ===")
    projects = list_projects_v3()

    print("=== Step 4: Relationship Map ===")
    print_relationship_map(vpcs, projects)
```

---

## 6. Quick Reference — cURL Cheat Sheet

```bash
PC="{pc_ip}"
AUTH="{username}:{password}"

# List VPCs
curl -sk -u "$AUTH" "https://$PC:9440/api/networking/v4.0/config/vpcs?\$limit=100" | python3 -m json.tool

# Get single VPC
curl -sk -u "$AUTH" "https://$PC:9440/api/networking/v4.0/config/vpcs/{vpcExtId}" | python3 -m json.tool

# List Projects (v3)
curl -sk -u "$AUTH" \
  -X POST "https://$PC:9440/api/nutanix/v3/projects/list" \
  -H "Content-Type: application/json" \
  -d '{"kind": "project", "length": 100, "offset": 0}' \
  | python3 -m json.tool
```

---

## Notes

- **Correct VPC path on PC 7.3.1.3:** `networking/v4.0` — not `v4.0.a1` (returns 404).
- **Projects v4 IAM** not available on PC 7.3.1.3 — use v3 fallback until PC is upgraded.
- **VPC↔Project join key:** `vpc_reference_list[].uuid` in v3 project response matches `extId` from the networking v4 VPC response.
- **`$limit` in bash** must be escaped as `\$limit` to avoid shell variable expansion.
- All mutating calls (POST / PUT / DELETE) require `Ntnx-Request-Id` with a fresh UUID v4.
- PUT and DELETE require `If-Match` with the ETag from a prior GET of the same resource.

---

## References

- [Nutanix Networking API Reference v4.0](https://developers.nutanix.com/api-reference?namespace=networking&version=v4.0)
- [Nutanix IAM API Reference](https://developers.nutanix.com/api-reference?namespace=iam&version=v4.1.b1)
- [Nutanix v3 API — Projects](https://developers.nutanix.com/api-reference?namespace=nutanix&version=v3)
- [Nutanix v4 API Namespaces](https://www.nutanix.dev/api-versions/)
- [ntnx-api-python-clients on GitHub](https://github.com/nutanix/ntnx-api-python-clients)
