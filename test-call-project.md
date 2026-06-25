# Nutanix API v4 — Test: VPC + Project Relationship

Tested against Prism Central 9440. Documents every call made, the version probe, and the results — with placeholders replacing real IPs and credentials.

---

## Environment

| Variable | Placeholder | Notes |
|---|---|---|
| Prism Central IP | `{pc_ip}` | FQDN or IPv4, no trailing slash |
| Port | `9440` | Default PC port |
| Username | `{username}` | PC local or directory account |
| Password | `{password}` | |

---

## Step 1 — Probe Which API Versions Are Available

Not all networking API versions are available on every PC build. Run this before any real call.

```bash
for ver in "v4.0.a1" "v4.0.b1" "v4.0.b2" "v4.0" "v4.1" "v4.1.a1" "v4.1.b1" "v4.2"; do
  code=$(curl -sk -u "{username}:{password}" \
    "https://{pc_ip}:9440/api/networking/$ver/config/vpcs?\$limit=1" \
    -o /dev/null -w "%{http_code}")
  echo "networking/$ver → HTTP $code"
done
```

**Result from this test:**

| Version | Status |
|---|---|
| `v4.0.a1` | ❌ 404 — not available |
| `v4.0.b1` | ✅ 200 |
| `v4.0.b2` | ✅ 200 |
| `v4.0` | ✅ 200 |
| `v4.1` | ✅ 200 |
| `v4.1.a1` | ❌ 404 |
| `v4.1.b1` | ❌ 404 |
| `v4.2` | ❌ 404 |

**Use `v4.0.b1`** — earliest confirmed GA on this cluster.

---

## Step 2 — List VPCs

```bash
curl -sk -u "{username}:{password}" \
  "https://{pc_ip}:9440/api/networking/v4.0.b1/config/vpcs?\$limit=100" \
  | python3 -m json.tool
```

**Key fields in response:**

| Field | Description |
|---|---|
| `data[].extId` | VPC UUID — used as reference in projects |
| `data[].name` | VPC display name |
| `data[].vpcType` | `REGULAR` (NAT) or `TRANSIT` |
| `data[].externalSubnets[].subnetReference` | External subnet UUID |
| `data[].externalSubnets[].externalIps` | SNAT IP pool |
| `data[].externalSubnets[].activeGatewayNodes` | HA gateway node IPs |
| `data[].commonDhcpOptions.domainNameServers` | DNS servers for all overlay subnets |

**Result from this test — 1 VPC found:**

| Field | Value |
|---|---|
| Name | `VPN-for-Test` |
| Type | `REGULAR` (NAT VPC) |
| DNS | `1.1.1.1` |
| External Subnet UUID | `f4c742b3-5ee8-421e-b069-0aa2bdbda2ad` |
| SNAT IPs | `10.8.23.23`, `10.8.23.24` *(redacted in your env)* |
| Gateway Nodes | 2 nodes (HA active-active) |

---

## Step 3 — Probe Projects API (v4 IAM)

Check if the v4 IAM projects endpoint is available on this PC.

```bash
for ver in "v4.0.b1" "v4.0.b2" "v4.0" "v4.1"; do
  code=$(curl -sk -u "{username}:{password}" \
    "https://{pc_ip}:9440/api/iam/$ver/authz/projects?\$limit=1" \
    -o /dev/null -w "%{http_code}")
  echo "iam/$ver → HTTP $code"
done
```

**Result:** All versions returned `404` — v4 IAM projects not available on this PC build. Use the v3 fallback below.

> v4 IAM projects (`iam/v4.x/authz/projects`) requires a newer PC release. v3 is supported until Q4-CY2026.

---

## Step 4 — List Projects (v3 Fallback)

```bash
curl -sk -u "{username}:{password}" \
  -X POST "https://{pc_ip}:9440/api/nutanix/v3/projects/list" \
  -H "Content-Type: application/json" \
  -d '{"kind": "project", "length": 100, "offset": 0}' \
  | python3 -m json.tool
```

**Key fields in response:**

| Field | Description |
|---|---|
| `entities[].metadata.uuid` | Project UUID |
| `entities[].status.name` | Project name |
| `entities[].status.state` | `COMPLETE` = healthy |
| `entities[].status.resources.vpc_reference_list` | VPCs attached to this project |
| `entities[].status.resources.subnet_reference_list` | Subnets available in project |
| `entities[].status.resources.default_subnet_reference` | Default subnet for new VMs |
| `entities[].status.resources.user_reference_list` | Users with access |
| `entities[].status.resources.external_user_group_reference_list` | AD/LDAP groups |
| `entities[].status.resources.cluster_reference_list` | Clusters scoped to project |

**Result from this test — 2 Projects found:**

### Project 1: `Project-VPC` — linked to VPC

| Field | Value |
|---|---|
| UUID | `e15d97b2-1b6b-43b3-829c-fa94f433a7a0` |
| State | `COMPLETE` |
| VPC | `VPN-for-Test` (via `vpc_reference_list`) |
| Default Subnet | `secodary-for-vpc-49` |
| Subnets | `overlay-49`, `secodary-for-vpc-49` |
| Users | `adminuser02@ntnxlab.local` |
| Created | 2026-06-15 |

### Project 2: `NTNX` — no VPC

| Field | Value |
|---|---|
| UUID | `e68b227a-8358-43cf-8697-8028e337b299` |
| State | `COMPLETE` |
| VPC | none |
| Default Subnet | `primary` |
| Subnets | `primary`, `secondary` |
| User Groups | `CN=SSP Admins,CN=Users,DC=ntnxlab,DC=local` |
| Created | 2026-06-12 |

---

## VPC ↔ Project Relationship Map

```
VPN-for-Test (VPC)
  └── Project-VPC
        ├── Subnets : overlay-49, secodary-for-vpc-49
        ├── Default : secodary-for-vpc-49 (external subnet = uplink)
        └── User    : adminuser02@ntnxlab.local
```

`NTNX` project uses flat VLAN subnets (`primary`, `secondary`) with no VPC attachment.

---

## Full Python Script

Runs all steps above in sequence: version probe → list VPCs → list projects → print relationship map.

```bash
pip install ntnx-networking-py-client requests urllib3
```

```python
"""
Nutanix API v4 — VPC + Project relationship mapper
Requires: ntnx-networking-py-client, requests, urllib3
"""

import sys
import requests
import urllib3
import ntnx_networking_py_client as networking

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

PC_IP     = "{pc_ip}"
PORT      = 9440
USERNAME  = "{username}"
PASSWORD  = "{password}"
BASE_URL  = f"https://{PC_IP}:{PORT}"
AUTH      = (USERNAME, PASSWORD)
HEADERS   = {"Content-Type": "application/json"}


# ── Step 1: probe networking API versions ────────────────────────────────────

def probe_networking_version():
    candidates = ["v4.0.b1", "v4.0.b2", "v4.0", "v4.1"]
    for ver in candidates:
        url = f"{BASE_URL}/api/networking/{ver}/config/vpcs?$limit=1"
        r = requests.get(url, auth=AUTH, verify=False)
        print(f"networking/{ver} → HTTP {r.status_code}")
        if r.status_code == 200:
            print(f"  ✅ Using: networking/{ver}\n")
            return ver
    print("No supported networking version found.")
    sys.exit(1)


# ── Step 2: list VPCs via SDK ────────────────────────────────────────────────

def list_vpcs(version):
    config = networking.Configuration()
    config.host       = PC_IP
    config.port       = PORT
    config.username   = USERNAME
    config.password   = PASSWORD
    config.verify_ssl = False

    client  = networking.ApiClient(configuration=config)
    api     = networking.VpcsApi(api_client=client)

    resp = api.list_vpcs(_page=0, _limit=100)
    vpcs = resp.data or []
    print(f"VPCs ({len(vpcs)} found):")
    for vpc in vpcs:
        ext_subnets = vpc.external_subnets or []
        snat_ips = []
        for es in ext_subnets:
            for ip in (es.external_ips or []):
                if ip.ipv4:
                    snat_ips.append(ip.ipv4.value)
        print(f"  • {vpc.name}  extId={vpc.ext_id}  type={vpc.vpc_type}  snat={snat_ips}")
    print()
    return vpcs


# ── Step 3: probe v4 IAM projects ────────────────────────────────────────────

def probe_iam_projects():
    for ver in ["v4.0.b1", "v4.0.b2", "v4.0", "v4.1"]:
        url = f"{BASE_URL}/api/iam/{ver}/authz/projects?$limit=1"
        r = requests.get(url, auth=AUTH, verify=False)
        print(f"iam/{ver} → HTTP {r.status_code}")
        if r.status_code == 200:
            return ver
    print("  ⚠️  v4 IAM projects not available — falling back to v3\n")
    return None


# ── Step 4: list projects via v3 ─────────────────────────────────────────────

def list_projects_v3():
    url  = f"{BASE_URL}/api/nutanix/v3/projects/list"
    body = {"kind": "project", "length": 100, "offset": 0}
    r    = requests.post(url, auth=AUTH, json=body, verify=False)
    r.raise_for_status()
    projects = r.json().get("entities", [])
    print(f"Projects ({len(projects)} found):")
    for p in projects:
        name     = p["status"]["name"]
        uuid     = p["metadata"]["uuid"]
        vpc_refs = p["status"]["resources"].get("vpc_reference_list", [])
        subnets  = [s["name"] for s in p["status"]["resources"].get("subnet_reference_list", [])]
        users    = [u["name"] for u in p["status"]["resources"].get("user_reference_list", [])]
        vpc_ids  = [v["uuid"] for v in vpc_refs]
        print(f"  • {name}  uuid={uuid}")
        print(f"    vpcs    : {vpc_ids or 'none'}")
        print(f"    subnets : {subnets}")
        print(f"    users   : {users or 'none'}")
    print()
    return projects


# ── Step 5: print relationship map ───────────────────────────────────────────

def print_relationship_map(vpcs, projects):
    vpc_lookup = {v.ext_id: v.name for v in vpcs}
    print("── VPC ↔ Project map ──────────────────────────────")
    for p in projects:
        name     = p["status"]["name"]
        vpc_refs = p["status"]["resources"].get("vpc_reference_list", [])
        if not vpc_refs:
            print(f"  {name}  →  (no VPC)")
            continue
        for ref in vpc_refs:
            vpc_name = vpc_lookup.get(ref["uuid"], ref["uuid"])
            subnets  = [s["name"] for s in p["status"]["resources"].get("subnet_reference_list", [])]
            default  = p["status"]["resources"].get("default_subnet_reference", {}).get("uuid", "")
            print(f"  {vpc_name}  →  {name}")
            print(f"    subnets : {subnets}")
            print(f"    default : {default}")


# ── Main ─────────────────────────────────────────────────────────────────────

if __name__ == "__main__":
    print("=== Step 1: Probe networking API version ===")
    net_ver = probe_networking_version()

    print("=== Step 2: List VPCs ===")
    vpcs = list_vpcs(net_ver)

    print("=== Step 3: Probe v4 IAM projects ===")
    probe_iam_projects()

    print("=== Step 4: List Projects (v3) ===")
    projects = list_projects_v3()

    print("=== Step 5: Relationship Map ===")
    print_relationship_map(vpcs, projects)
```

---

## Quick Reference — cURL Cheat Sheet

```bash
PC="{pc_ip}"
AUTH="{username}:{password}"

# List VPCs
curl -sk -u "$AUTH" "https://$PC:9440/api/networking/v4.0.b1/config/vpcs?\$limit=100" | python3 -m json.tool

# Get single VPC
curl -sk -u "$AUTH" "https://$PC:9440/api/networking/v4.0.b1/config/vpcs/{vpcExtId}" | python3 -m json.tool

# List Projects (v3)
curl -sk -u "$AUTH" \
  -X POST "https://$PC:9440/api/nutanix/v3/projects/list" \
  -H "Content-Type: application/json" \
  -d '{"kind": "project", "length": 100, "offset": 0}' \
  | python3 -m json.tool
```

---

## Notes

- **v4 IAM projects** (`iam/v4.x/authz/projects`) was not available on the tested PC build — always probe first and fall back to v3 if needed.
- **VPC UUID** in `vpc_reference_list` matches `extId` from the networking API — use it to join the two datasets.
- **`$limit` dollar sign** must be escaped as `\$limit` in bash to avoid shell variable expansion.
- All mutating calls (POST / PUT / DELETE) require the `Ntnx-Request-Id` header with a UUID value.
