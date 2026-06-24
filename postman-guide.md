# Testing Nutanix API v4 with Postman

## 1. Disable SSL Verification (for lab/self-signed cert)

Postman → Settings → **General** → toggle off **SSL certificate verification**

---

## 2. Create an Environment

Postman → **Environments** → New → name it `Nutanix Lab`

| Variable | Value |
|---|---|
| `pc_ip` | `10.0.0.1` (your Prism Central IP) |
| `username` | `admin` |
| `password` | `your_password` |

Save, then select `Nutanix Lab` from the environment dropdown (top-right).

---

## 3. Set Up Basic Auth

In your request → **Authorization** tab:

- Type: **Basic Auth**
- Username: `{{username}}`
- Password: `{{password}}`

---

## 4. List All VMs

**Method:** `GET`

**URL:**
```
https://{{pc_ip}}:9440/api/vmm/v4.1/ahv/config/vms
```

**Query Params** (Params tab):

| Key | Value |
|---|---|
| `$page` | `0` |
| `$limit` | `100` |
| `$select` | `extId,name,numSockets,numCoresPerSocket,memorySizeBytes,disks` |

Click **Send** → you should get `200 OK` with a JSON body.

---

## 5. Read the Response

Look for each VM object in `data[]`:

```json
{
  "data": [
    {
      "extId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
      "name": "my-vm",
      "numSockets": 2,
      "numCoresPerSocket": 4,
      "memorySizeBytes": 17179869184,
      "disks": [
        {
          "diskSizeBytes": 107374182400,
          "diskAddress": {
            "busType": "SCSI",
            "index": 0
          }
        }
      ]
    }
  ]
}
```

**Calculate from the response:**

| Metric | Formula | Example |
|---|---|---|
| Total vCPU | `numSockets × numCoresPerSocket` | `2 × 4 = 8` |
| RAM (GiB) | `memorySizeBytes ÷ 1073741824` | `17179869184 ÷ 1073741824 = 16 GiB` |
| Disk (GiB) | `diskSizeBytes ÷ 1073741824` | `107374182400 ÷ 1073741824 = 100 GiB` |

---

## 6. Filter VMs by Name (Optional)

Add `$filter` to query params:

| Key | Value |
|---|---|
| `$filter` | `name eq 'my-vm'` |

---

## 7. Get a Single VM by UUID

**Method:** `GET`

**URL:**
```
https://{{pc_ip}}:9440/api/vmm/v4.1/ahv/config/vms/{extId}
```

Replace `{extId}` with the UUID from step 5. This returns full VM details including all disks, NICs, and boot config.

---

## 8. Paginate (more than 100 VMs)

Increment `$page` until `data` is empty:

| Request | `$page` |
|---|---|
| First batch | `0` |
| Second batch | `1` |
| Stop when | `data: []` |

---

## 9. Save as a Collection

- Click **Save** → New Collection → name it `Nutanix API v4`
- Save each request (List VMs, Get VM) under the collection
- Share the collection via **Export** → JSON → commit to this repo
