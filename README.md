# Nutanix API v4 — Test & Reference

Hands-on test scripts and reference docs for the Nutanix REST API v4, built and verified against a live Prism Central environment. Covers VPC, Projects, and VM information retrieval.

---

> **⚠️ Version Disclaimer**
>
> All code, endpoints, and test results in this repository were verified on the following environment. API namespace availability, endpoint paths, and response schemas **vary across PC and AOS releases**. What works on one build may return `404` or behave differently on another.
>
> **Tested Environment**
>
> | Component | Tested Version | Notes |
> |---|---|---|
> | Prism Central | `pc.7.3.1.3` | `el8.5-release-ganges-7.3.1.3` |
> | AOS (RNO-POC012) | `7.3` | Primary PE cluster |
> | AOS (DR) | `7.3` | DR PE cluster |
> | PC Nodes | 1 | Single-node PC |
> | Storage Type | All-Flash | |
> | Architecture | X86_64 | |
>
> **API Version Availability (confirmed on this build)**
>
> | Namespace | Working Versions | Not Available |
> |---|---|---|
> | `networking` | `v4.0.b1`, `v4.0.b2`, `v4.0`, `v4.1` | `v4.0.a1`, `v4.1.a1`+ |
> | `vmm` | `v4.0.b1` | `v4.0.b2`, `v4.0`, `v4.1` |
> | `clustermgmt` | `v4.0.b1` | — |
> | `prism` (tasks) | `v4.0.b1` | — |
> | `iam` v4 projects | ❌ Not available — returns 404 | Use v3 fallback |
> | v3 API (`/api/nutanix/v3/`) | ✅ Fully available | — |
> | v1 REST (`/PrismGateway/services/rest/v1/`) | ✅ Fully available | — |
>
> **Before running any script:** use the version probe steps in each test file to confirm which API versions your cluster supports. Never hardcode a version without probing first. Your environment may differ even on the same PC release depending on the upgrade path taken.

---

## What's in This Repo

| File | What it covers |
|---|---|
| [vpc-and-projects-api.md](./vpc-and-projects-api.md) | VPC full CRUD (List, Get, Create, Update, Delete) + Projects |
| [test-call-project.md](./test-call-project.md) | Live test: VPC list → Project list → VPC↔Project relationship map |
| [test-call-vm-info.md](./test-call-vm-info.md) | Live test: VM name, IP, CPU, Memory, Disk size/usage, Network, Power state |
| [postman-guide.md](./postman-guide.md) | Postman collection setup and quick test steps |

---

## Nutanix API Overview

Nutanix exposes infrastructure management through a family of versioned REST APIs under **Prism Central** (port 9440). Each functional area has its own **namespace**.

### Base URL Pattern

```
https://{prism_central_ip}:9440/api/{namespace}/{version}/{resource}
```

### Namespaces Used in This Repo

| Namespace | Purpose | Confirmed Version |
|---|---|---|
| `vmm` | VM config — list, create, update VMs | `v4.0.b1` |
| `networking` | VPC, subnets, virtual switches | `v4.0.b1` |
| `iam` | Projects, users, roles (v4 not yet GA on all builds) | v3 fallback |
| `prism` | Task polling after async operations | `v4.0.b1` |

> Version availability depends on your PC build. Always probe before using — see the version probe pattern in each test file.

### Authentication

All calls use **HTTP Basic Auth**. For automation, Nutanix recommends an **API Key** via the `X-Ntnx-Api-Key` header using a service account.

```bash
# Basic Auth
curl -sk -u "{username}:{password}" "https://{pc_ip}:9440/api/..."

# API Key
curl -sk -H "X-Ntnx-Api-Key: {api_key}" "https://{pc_ip}:9440/api/..."
```

### Required Headers for Mutating Calls

POST, PUT, and DELETE requests require:

| Header | Value | Notes |
|---|---|---|
| `Content-Type` | `application/json` | Always |
| `Ntnx-Request-Id` | UUID v4 | Ensures idempotency — generate fresh per request |
| `If-Match` | ETag from prior GET | Required for PUT and DELETE only |

### Pagination

All list endpoints use OData-style query parameters:

| Parameter | Default | Max | Description |
|---|---|---|---|
| `$page` | `0` | — | Zero-based page number |
| `$limit` | `50` | `100` | Results per page |
| `$filter` | — | — | OData filter e.g. `name eq 'my-vpc'` |
| `$orderby` | — | — | e.g. `name asc` |

Always loop pages until `data` is empty — the API caps at 100 per request.

### Async Operations

Create, Update, and Delete return a **task extId**, not the resource. Poll the task until it reaches a terminal state:

```bash
curl -sk -u "{username}:{password}" \
  "https://{pc_ip}:9440/api/prism/v4.0.b1/config/tasks/{taskExtId}" \
  | python3 -m json.tool
```

Terminal statuses: `SUCCEEDED`, `FAILED`, `CANCELLED`.

---

## Test Results Summary

All tests run against a live Prism Central cluster.

### Test 1 — VPC & Projects (`test-call-project.md`, `vpc-and-projects-api.md`)

| Step | API | Result |
|---|---|---|
| Version probe | `networking/v4.x` | `v4.0.b1` → `v4.1` ✅ / `v4.0.a1` ❌ 404 |
| List VPCs | `GET /api/networking/v4.0/config/vpcs` | 1 VPC: **VPN-for-Test** (`extId: 71f5d1e9-...`) |
| VPC type | — | REGULAR (NAT), SNAT IPs: `10.8.23.23/24`, DNS: `1.1.1.1` |
| v4 Projects probe | `iam/v4.x/authz/projects` | All versions 404 — not available on PC 7.3.1.3 |
| List Projects (v3) | `POST /api/nutanix/v3/projects/list` | 2 projects: **Project-VPC**, **NTNX** |
| VPC↔Project link | via `vpc_reference_list` | `Project-VPC` → `VPN-for-Test` ✅ confirmed |

### Test 2 — VM Information (`test-call-vm-info.md`)

| Step | API | Result |
|---|---|---|
| Version probe | `vmm/v4.x/ahv/config/vms` | `v4.0.b1` ✅ / `v4.0.b2`+ ❌ |
| List VMs | `GET /api/vmm/v4.0.b1/ahv/config/vms` | 48 VMs found |
| VM config | v4 VMM | Name, CPU, RAM, Disk size, NIC, IP, Power state |
| Live stats | `GET /PrismGateway/services/rest/v1/vms/{uuid}` | CPU %, Mem %, IOPS, Disk used, Net RX/TX |
| v4 Stats endpoint | `GET /api/vmm/v4.0.b1/ahv/stats/vms/{extId}` | Requires `$select` — errors on this build, use v1 |

---

## Prerequisites

- Prism Central **pc.2024.3+** / AOS **7.0+**
- Python **3.8+**

```bash
pip install ntnx-vmm-py-client ntnx-networking-py-client ntnx-prism-py-client requests urllib3 tabulate
```

---

## Quick Start

```bash
PC="{pc_ip}"
AUTH="{username}:{password}"

# List VPCs
curl -sk -u "$AUTH" \
  "https://$PC:9440/api/networking/v4.0.b1/config/vpcs?\$limit=100" \
  | python3 -m json.tool

# List Projects (v3)
curl -sk -u "$AUTH" \
  -X POST "https://$PC:9440/api/nutanix/v3/projects/list" \
  -H "Content-Type: application/json" \
  -d '{"kind":"project","length":100,"offset":0}' \
  | python3 -m json.tool

# List VMs
curl -sk -u "$AUTH" \
  "https://$PC:9440/api/vmm/v4.0.b1/ahv/config/vms?\$limit=100" \
  | python3 -m json.tool

# Get VM live stats
curl -sk -u "$AUTH" \
  "https://$PC:9440/PrismGateway/services/rest/v1/vms/{vmUuid}" \
  | python3 -m json.tool
```

---

## Known Limitations (on tested PC build)

| Limitation | Workaround |
|---|---|
| `networking/v4.0.a1` not available | Use `v4.0.b1` or `v4.0` |
| `iam/v4.x/authz/projects` returns 404 on PC 7.3.1.3 | Use v3: `POST /api/nutanix/v3/projects/list` |
| `vmm/v4.0.b2`+ not available | Use `vmm/v4.0.b1` |
| v4 VM stats endpoint errors without `$select` | Use v1: `/PrismGateway/services/rest/v1/vms/{uuid}` |
| Stats metrics with value `-1` | Not available — VM may be off or NGT not installed |

---

## References

- [Nutanix v4 API User Guide](https://www.nutanix.dev/nutanix-api-user-guide/)
- [API Namespace Reference](https://developers.nutanix.com/api-reference?namespace=networking&version=v4.0.a1)
- [VMM API Reference](https://developers.nutanix.com/api-reference?namespace=vmm&version=v4.0.b1)
- [Prism API Reference (task polling)](https://developers.nutanix.com/api-reference?namespace=prism&version=v4.0.b1)
- [ntnx-api-python-clients on GitHub](https://github.com/nutanix/ntnx-api-python-clients)
- [All API versions](https://www.nutanix.dev/api-versions/)
