# Nutanix API v4 — VPC CRUD + Projects (Prism Central 7.3 / AOS 7.0+)

## Namespace Overview

| Resource | Namespace | SDK Package | GA in PC |
|---|---|---|---|
| VPC (Flow Virtual Networking) | `networking` | `ntnx-networking-py-client` | pc.2024.3 / AOS 7.0 |
| Projects | `iam` (authz) | `ntnx-iam-py-client` | pc.2024.3+ |

> **Note:** Projects in v4 live under the **IAM** namespace (`iam/v4/authz/projects`), not the `prism` namespace. The `prism` namespace covers tasks and categories only. If a projects endpoint is not yet available for your PC version, fall back to v3: `POST /api/nutanix/v3/projects/list` — v3 remains supported until Q4-CY2026.

---

## VPC API — Full CRUD

Base path: `https://{pc_ip}:9440/api/networking/v4.0.a1/config/vpcs`

| Operation | Method | Path |
|---|---|---|
| List VPCs | `GET` | `/vpcs` |
| Get VPC | `GET` | `/vpcs/{extId}` |
| Create VPC | `POST` | `/vpcs` |
| Update VPC | `PUT` | `/vpcs/{extId}` |
| Delete VPC | `DELETE` | `/vpcs/{extId}` |

**Headers required on every mutating request (POST / PUT / DELETE):**

| Header | Value |
|---|---|
| `Content-Type` | `application/json` |
| `Ntnx-Request-Id` | A fresh UUID v4 — ensures idempotency |
| `If-Match` | Current ETag of the resource (PUT / DELETE only — fetch it first with GET) |

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
config.host = "10.0.0.1"
config.port = 9440
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

---

## 2. Get VPC by extId

### Endpoint

```
GET https://{pc_ip}:9440/api/networking/v4.0.a1/config/vpcs/{extId}
```

### cURL Example

```bash
curl -sk -u admin:password \
  "https://10.0.0.1:9440/api/networking/v4.0.a1/config/vpcs/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" \
  | python3 -m json.tool
```

> The `ETag` header in the response is required for subsequent PUT or DELETE calls.

### Python SDK

```python
vpc = api.get_vpc_by_id(ext_id="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx")
print(vpc.data.name)
```

---

## 3. Create VPC

### Endpoint

```
POST https://{pc_ip}:9440/api/networking/v4.0.a1/config/vpcs
```

### Request Body (minimal — no external subnet / no-NAT VPC)

```json
{
  "name": "dev-vpc",
  "description": "Development VPC"
}
```

### Request Body (with external subnet — NAT VPC)

```json
{
  "name": "prod-vpc",
  "description": "Production VPC with NAT",
  "externalSubnets": [
    {
      "subnetReference": "yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy"
    }
  ],
  "commonDhcpOptions": {
    "domainNameServers": [
      { "ipv4": { "value": "8.8.8.8", "prefixLength": 32 } },
      { "ipv4": { "value": "8.8.4.4", "prefixLength": 32 } }
    ]
  }
}
```

> **`externalSubnets.subnetReference`** is the `extId` of an existing external subnet (marked `isExternal: true`). Omitting this creates a no-NAT (isolated) VPC.

### cURL Example

```bash
curl -sk -u admin:password \
  -X POST "https://10.0.0.1:9440/api/networking/v4.0.a1/config/vpcs" \
  -H "Content-Type: application/json" \
  -H "Ntnx-Request-Id: $(python3 -c 'import uuid; print(uuid.uuid4())')" \
  -d '{
    "name": "prod-vpc",
    "description": "Production VPC with NAT",
    "externalSubnets": [
      { "subnetReference": "yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy" }
    ]
  }' \
  | python3 -m json.tool
```

### Response

Returns a **task** `extId` — poll the task until it reaches `SUCCEEDED`:

```json
{
  "data": {
    "extId": "ZXJnb24jbWVkaXVtX3Rhc2tzIzE..."
  }
}
```

### Poll the Task

```bash
curl -sk -u admin:password \
  "https://10.0.0.1:9440/api/prism/v4.0.b1/config/tasks/{taskExtId}" \
  | python3 -m json.tool
```

Task terminal statuses: `SUCCEEDED`, `FAILED`, `CANCELLED`.

### Python SDK

```python
import uuid
import time
import ntnx_networking_py_client as networking
import ntnx_networking_py_client.models.networking.v4.config as v4config
import ntnx_prism_py_client as prism

# --- networking client setup ---
net_config = networking.Configuration()
net_config.host = "10.0.0.1"
net_config.port = 9440
net_config.username = "admin"
net_config.password = "password"
net_config.verify_ssl = False
net_client = networking.ApiClient(configuration=net_config)
vpcs_api = networking.VpcsApi(api_client=net_client)

# --- prism client for task polling ---
prism_config = prism.Configuration()
prism_config.host = "10.0.0.1"
prism_config.port = 9440
prism_config.username = "admin"
prism_config.password = "password"
prism_config.verify_ssl = False
prism_client = prism.ApiClient(configuration=prism_config)
tasks_api = prism.TasksApi(api_client=prism_client)

# --- build the VPC body ---
new_vpc = v4config.Vpc(
    name="prod-vpc",
    description="Production VPC with NAT",
    external_subnets=[
        v4config.ExternalSubnet(
            subnet_reference="yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy"
        )
    ]
)

# --- create ---
resp = vpcs_api.create_vpc(
    body=new_vpc,
    async_req=False,
    ntnx_request_id=str(uuid.uuid4())
)
task_ext_id = resp.data.ext_id
print(f"Task created: {task_ext_id}")

# --- poll until done ---
while True:
    task = tasks_api.get_task_by_id(ext_id=task_ext_id)
    status = task.data.status.value
    print(f"Task status: {status}")
    if status in ("SUCCEEDED", "FAILED", "CANCELLED"):
        break
    time.sleep(3)
```

---

## 4. Update VPC

### Endpoint

```
PUT https://{pc_ip}:9440/api/networking/v4.0.a1/config/vpcs/{extId}
```

**You must include the `If-Match` header** with the ETag value from a prior GET.

### cURL Example

```bash
# Step 1: fetch current ETag
ETAG=$(curl -sk -u admin:password -I \
  "https://10.0.0.1:9440/api/networking/v4.0.a1/config/vpcs/{extId}" \
  | grep -i 'etag' | awk '{print $2}' | tr -d '\r')

# Step 2: PUT with If-Match
curl -sk -u admin:password \
  -X PUT "https://10.0.0.1:9440/api/networking/v4.0.a1/config/vpcs/{extId}" \
  -H "Content-Type: application/json" \
  -H "Ntnx-Request-Id: $(python3 -c 'import uuid; print(uuid.uuid4())')" \
  -H "If-Match: ${ETAG}" \
  -d '{
    "name": "prod-vpc-updated",
    "description": "Updated description"
  }' \
  | python3 -m json.tool
```

---

## 5. Delete VPC

### Endpoint

```
DELETE https://{pc_ip}:9440/api/networking/v4.0.a1/config/vpcs/{extId}
```

**Requires `If-Match` ETag header** (same pattern as PUT).

### cURL Example

```bash
ETAG=$(curl -sk -u admin:password -I \
  "https://10.0.0.1:9440/api/networking/v4.0.a1/config/vpcs/{extId}" \
  | grep -i 'etag' | awk '{print $2}' | tr -d '\r')

curl -sk -u admin:password \
  -X DELETE "https://10.0.0.1:9440/api/networking/v4.0.a1/config/vpcs/{extId}" \
  -H "Ntnx-Request-Id: $(python3 -c 'import uuid; print(uuid.uuid4())')" \
  -H "If-Match: ${ETAG}" \
  | python3 -m json.tool
```

Returns a task extId — poll until `SUCCEEDED`.

---

## 6. List Projects

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
config.host = "10.0.0.1"
config.port = 9440
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

## 7. v3 Fallback (if v4 projects endpoint not yet available)

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

## 8. Postman — Quick Test Steps

### VPC List

1. `GET https://{{pc_ip}}:9440/api/networking/v4.0.a1/config/vpcs`
2. Authorization: Basic Auth → `{{username}}` / `{{password}}`
3. Params: `$limit=100`

### VPC Create

1. `POST https://{{pc_ip}}:9440/api/networking/v4.0.a1/config/vpcs`
2. Authorization: Basic Auth
3. Headers: `Content-Type: application/json`, `Ntnx-Request-Id: {{$guid}}`
4. Body (raw JSON):
   ```json
   {
     "name": "test-vpc",
     "description": "Created via Postman"
   }
   ```

### VPC Delete

1. `DELETE https://{{pc_ip}}:9440/api/networking/v4.0.a1/config/vpcs/{{vpcExtId}}`
2. Authorization: Basic Auth
3. Headers: `Ntnx-Request-Id: {{$guid}}`, `If-Match: {{etag}}`
4. Get ETag first: `GET` the VPC and copy the `ETag` response header → set as `{{etag}}` variable

### Project List (v4)

1. `GET https://{{pc_ip}}:9440/api/iam/v4.0/authz/projects`
2. Authorization: Basic Auth
3. Params: `$limit=100`

### Project List (v3 fallback)

1. `POST https://{{pc_ip}}:9440/api/nutanix/v3/projects/list`
2. Authorization: Basic Auth
3. Body (raw JSON): `{"kind": "project", "length": 100, "offset": 0}`

---

## SDK Install (all packages)

```bash
pip install ntnx-networking-py-client ntnx-iam-py-client ntnx-prism-py-client
```

---

## References

- [Nutanix Networking API Reference (v4.0.a1)](https://developers.nutanix.com/api-reference?namespace=networking&version=v4.0.a1)
- [Nutanix IAM API Reference](https://developers.nutanix.com/api-reference?namespace=iam&version=v4.1.b1)
- [Nutanix Prism API Reference (task polling)](https://developers.nutanix.com/api-reference?namespace=prism&version=v4.0.b1)
- [Nutanix v4 API Namespaces (all versions)](https://www.nutanix.dev/api-versions/)
- [ntnx-api-python-clients on GitHub](https://github.com/nutanix/ntnx-api-python-clients)
- [Nutanix v4 API User Guide](https://www.nutanix.dev/nutanix-api-user-guide/)
- [Flow Virtual Networking — VPC concepts](https://portal.nutanix.com/page/documents/details?targetId=Nutanix-Flow-Virtual-Networking-Guide-vpc_2024_2:ear-flow-nw-vpc-concepts-pc-c.html)
