# Nutanix API v4 — List VPC and Projects (Prism Central 7.3.x)

## Namespace Overview

| Resource | Namespace | SDK Package | GA in PC |
|---|---|---|---|
| VPC (Flow Virtual Networking) | `networking` | `ntnx-networking-py-client` | pc.2024.3 / AOS 7.0 |
| Projects | `iam` (authz) | `ntnx-iam-py-client` | pc.2024.3+ |

> **Note:** Projects in v4 live under the **IAM** namespace (`iam/v4/authz/projects`), not the `prism` namespace. The `prism` namespace covers tasks and categories only. If a projects endpoint is not yet available for your PC version, fall back to v3: `POST /api/nutanix/v3/projects/list` — v3 remains supported until Q4-CY2026.

---

## 1. List VPCs

### Endpoint

```
GET https://{pc_ip}:9440/api/networking/v4.0.a1/config/vpcs
```

### Query Parameters

| Parameter | Type | Default | Notes |
|---|---|---|---|
| `$page` | int | 0 | Zero-based page number |
| `$limit` | int | 50 | Max 100 per request |
| `$filter` | string | — | OData v4.01 — e.g. `name eq 'my-vpc'` |
| `$orderby` | string | — | e.g. `name asc` |

### cURL Example

```bash
curl -sk -u admin:password \
  "https://10.0.0.1:9440/api/networking/v4.0.a1/config/vpcs?$limit=100" \
  | python3 -m json.tool
```

### Key Response Fields

| Field | Type | Description |
|---|---|---|
| `extId` | string | VPC UUID |
| `name` | string | VPC display name |
| `externalSubnets` | array | Uplink subnets for external connectivity |
| `externalRoutingDomainRef` | object | Routing domain reference |
| `commonDhcpOptions` | object | DHCP DNS/NTP settings shared across subnets |
| `snatIps` | array | SNAT IP pool (NAT VPCs only) |

### Example Response

```json
{
  "data": [
    {
      "extId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
      "name": "prod-vpc",
      "externalSubnets": [
        {
          "subnetReference": "yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy",
          "externalIps": ["192.168.10.5"],
          "gatewayNodes": []
        }
      ],
      "commonDhcpOptions": {
        "domainNameServers": [
          { "ipv4": { "value": "8.8.8.8" } }
        ]
      }
    }
  ],
  "metadata": {
    "totalAvailableResults": 3,
    "page": 0,
    "limit": 100
  }
}
```

### Python SDK

```python
import ntnx_networking_py_client as networking

config = networking.Configuration()
config.host = "https://10.0.0.1:9440"
config.username = "admin"
config.password = "password"
config.verify_ssl = False

client = networking.ApiClient(configuration=config)
api = networking.VpcsApi(api_client=client)

page = 0
all_vpcs = []
while True:
    resp = api.list_vpcs(_page=page, _limit=100)
    vpcs = resp.data
    if not vpcs:
        break
    all_vpcs.extend(vpcs)
    page += 1

for vpc in all_vpcs:
    print(f"{vpc.name} — extId: {vpc.ext_id}")
```

### SDK Install

```bash
pip install ntnx-networking-py-client
```

---

## 2. List Projects

> Projects are managed under the **IAM** namespace in v4. The endpoint path is `iam/v4.0/authz/projects` (RC in PC 7.3.x — verify availability on your cluster before using; fall back to v3 if not yet exposed).

### Endpoint

```
GET https://{pc_ip}:9440/api/iam/v4.0/authz/projects
```

### Query Parameters

Same OData pattern as other v4 namespaces:

| Parameter | Type | Notes |
|---|---|---|
| `$page` | int | Zero-based |
| `$limit` | int | Max 100 |
| `$filter` | string | e.g. `name eq 'dev-project'` |
| `$select` | string | e.g. `extId,name,description` |

### cURL Example

```bash
curl -sk -u admin:password \
  "https://10.0.0.1:9440/api/iam/v4.0/authz/projects?$limit=100" \
  | python3 -m json.tool
```

### Key Response Fields

| Field | Type | Description |
|---|---|---|
| `extId` | string | Project UUID |
| `name` | string | Project name |
| `description` | string | Project description |
| `userList` | array | Users assigned to the project |
| `userGroupList` | array | User groups assigned |
| `subnetReferenceList` | array | Subnets available to the project |
| `clusterReferenceList` | array | Clusters scoped to the project |
| `defaultSubnetReference` | object | Default subnet for new VMs |

### Python SDK

```python
import ntnx_iam_py_client as iam

config = iam.Configuration()
config.host = "https://10.0.0.1:9440"
config.username = "admin"
config.password = "password"
config.verify_ssl = False

client = iam.ApiClient(configuration=config)
api = iam.ProjectsApi(api_client=client)

page = 0
all_projects = []
while True:
    resp = api.list_projects(_page=page, _limit=100)
    projects = resp.data
    if not projects:
        break
    all_projects.extend(projects)
    page += 1

for p in all_projects:
    print(f"{p.name} — extId: {p.ext_id}")
```

### SDK Install

```bash
pip install ntnx-iam-py-client
```

---

## 3. v3 Fallback (if v4 projects endpoint not yet available)

If `iam/v4.0/authz/projects` returns 404 on your PC 7.3.x build, use v3 — it is supported until Q4-CY2026.

```bash
curl -sk -u admin:password \
  -X POST "https://10.0.0.1:9440/api/nutanix/v3/projects/list" \
  -H "Content-Type: application/json" \
  -d '{"kind": "project", "length": 100, "offset": 0}' \
  | python3 -m json.tool
```

Key v3 response fields: `metadata.uuid`, `spec.name`, `spec.description`, `spec.resources.subnet_reference_list`, `spec.resources.cluster_reference_list`.

---

## 4. Postman — Quick Test Steps

### VPC List

1. `GET https://{{pc_ip}}:9440/api/networking/v4.0.a1/config/vpcs`
2. Authorization: Basic Auth → `{{username}}` / `{{password}}`
3. Params: `$limit=100`

### Project List (v4)

1. `GET https://{{pc_ip}}:9440/api/iam/v4.0/authz/projects`
2. Authorization: Basic Auth
3. Params: `$limit=100`

### Project List (v3 fallback)

1. `POST https://{{pc_ip}}:9440/api/nutanix/v3/projects/list`
2. Authorization: Basic Auth
3. Body (raw JSON): `{"kind": "project", "length": 100, "offset": 0}`

---

## References

- [Nutanix Networking API Reference](https://developers.nutanix.com/api-reference?namespace=networking&version=v4.0.a1)
- [Nutanix IAM API Reference](https://developers.nutanix.com/api-reference?namespace=iam&version=v4.1.b1)
- [Nutanix v4 API Namespaces (all versions)](https://www.nutanix.dev/api-versions/)
- [ntnx-api-python-clients on GitHub](https://github.com/nutanix/ntnx-api-python-clients)
- [Flow Virtual Networking — VPC concepts](https://portal.nutanix.com/page/documents/details?targetId=Nutanix-Flow-Virtual-Networking-Guide-vpc_2024_2:ear-flow-nw-vpc-concepts-pc-c.html)
