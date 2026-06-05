# Azure SFTP POC — Full Setup Guide
### HNS + SFTP + ACL Isolation with SSH Key Authentication

---

## Prerequisites

Before starting, ensure you have:
- Azure CLI installed and logged in (`az login`)
- `sshpass` installed (`brew install sshpass`)
- Access to an Azure subscription

---

## Variables — Set These First

```bash
SUB_ID=$(az account show --query id -o tsv)
MY_USER=$(az ad signed-in-user show --query id -o tsv)
RESOURCE_GROUP="rg-sftp-poc"
STORAGE_ACCOUNT="stsftpprod001"
CONTAINER="sftpcontainer"
LOCATION="USGovTexas"
SFTP_USER_A="sftpusera"
SFTP_USER_B="sftpuserb"

echo "================================================"
echo "Subscription  : $SUB_ID"
echo "Resource Group: $RESOURCE_GROUP"
echo "Storage Acct  : $STORAGE_ACCOUNT"
echo "My User ID    : $MY_USER"
echo "================================================"
```

---

## Step 1 — Create Resource Group

```bash
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION \
  --query "properties.provisioningState" \
  -o tsv
# Expected: Succeeded
```

---

## Step 2 — Create Storage Account with HNS + SFTP Enabled

> **Important:** `--allow-shared-key-access` is intentionally omitted.
> SSH key authentication does not require shared key access.

```bash
az storage account create \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --sku Standard_LRS \
  --kind StorageV2 \
  --enable-hierarchical-namespace true \
  --enable-sftp true \
  --enable-local-user true \
  --allow-blob-public-access false \
  --min-tls-version TLS1_2
```

### Verify All Settings

```bash
az storage account show \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --query "{HNS:isHnsEnabled, SFTP:isSftpEnabled, LocalUser:isLocalUserEnabled}" \
  -o json
```

**Expected:**
```json
{
  "HNS": true,
  "SFTP": true,
  "LocalUser": true
}
```

---

## Step 3 — Assign Storage Blob Data Owner Role

> Required for ACL management (POSIX permissions on directories).
> `Storage Blob Data Contributor` is not sufficient — it cannot set ACLs.

```bash
az role assignment create \
  --assignee $MY_USER \
  --role "Storage Blob Data Owner" \
  --scope $(az storage account show \
    --name $STORAGE_ACCOUNT \
    --resource-group $RESOURCE_GROUP \
    --query id -o tsv)

echo "Waiting 3 minutes for role propagation..."
sleep 180
echo "Done"
```

### Verify Role Assignment

```bash
az role assignment list \
  --assignee $MY_USER \
  --scope $(az storage account show \
    --name $STORAGE_ACCOUNT \
    --resource-group $RESOURCE_GROUP \
    --query id -o tsv) \
  --query "[].{Role:roleDefinitionName}" \
  -o table
# Expected: Storage Blob Data Owner
```

---

## Step 4 — Create Container and Home Directories

```bash
# Create container
az storage fs create \
  --name $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login

# Create home directory for User A
az storage fs directory create \
  --name "userA-home" \
  --file-system $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login

# Create home directory for User B
az storage fs directory create \
  --name "userB-home" \
  --file-system $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login
```

### Verify Directories

```bash
az storage fs directory list \
  --file-system $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login \
  -o table
# Expected: userA-home and userB-home listed
```

---

## Step 5 — Create SFTP Local Users

```bash
# Create User A — scoped to userA-home
az storage account local-user create \
  --account-name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --name $SFTP_USER_A \
  --home-directory "$CONTAINER/userA-home" \
  --has-ssh-password true \
  --permission-scope \
    permissions=rcwdl \
    service=blob \
    resource-name="$CONTAINER"

# Create User B — scoped to userB-home
az storage account local-user create \
  --account-name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --name $SFTP_USER_B \
  --home-directory "$CONTAINER/userB-home" \
  --has-ssh-password true \
  --permission-scope \
    permissions=rcwdl \
    service=blob \
    resource-name="$CONTAINER"
```

### Verify Users Created

```bash
az storage account local-user list \
  --account-name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --query "[].{Name:name, HomeDir:homeDirectory, HasPassword:hasSshPassword}" \
  -o table
```

**Expected:**

| Name      | HomeDir                      | HasPassword |
|-----------|------------------------------|-------------|
| sftpusera | sftpcontainer/userA-home     | True        |
| sftpuserb | sftpcontainer/userB-home     | True        |

> **Permission scope values explained:**
> | Letter | Meaning |
> |--------|---------|
> | `r` | Read / Download files |
> | `c` | Create / Upload new files |
> | `w` | Write / Overwrite files |
> | `d` | Delete files |
> | `l` | List directory contents |

---

## Step 6 — Generate SSH Key and Attach to Users

```bash
# Generate dedicated SSH key pair for this POC
ssh-keygen -t ecdsa -b 256 \
  -f ~/.ssh/sftp_poc_key \
  -N "" \
  -C "sftp-poc-test"

# View the public key
cat ~/.ssh/sftp_poc_key.pub
```

### Attach SSH Key to User A

```bash
az storage account local-user update \
  --account-name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --name $SFTP_USER_A \
  --ssh-authorized-key key="$(cat ~/.ssh/sftp_poc_key.pub)"
```

### Attach SSH Key to User B

```bash
az storage account local-user update \
  --account-name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --name $SFTP_USER_B \
  --ssh-authorized-key key="$(cat ~/.ssh/sftp_poc_key.pub)"
```

### Verify Keys Attached

```bash
az storage account local-user list \
  --account-name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --query "[].{Name:name, SSHKeys:sshAuthorizedKeys}" \
  -o json
```

---

## Step 7 — Get User SIDs for ACL Assignment

```bash
USER_A_OID=$(az storage account local-user show \
  --account-name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --name $SFTP_USER_A \
  --query "sid" -o tsv)

USER_B_OID=$(az storage account local-user show \
  --account-name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --name $SFTP_USER_B \
  --query "sid" -o tsv)

echo "User A SID: $USER_A_OID"
echo "User B SID: $USER_B_OID"
```

---

## Step 8 — Prove the Problem Exists (Before Fix)

Connect as User A and verify they can see and enter User B's directory — this is the security gap we are fixing.

```bash
sftp \
  -i ~/.ssh/sftp_poc_key \
  -o IdentitiesOnly=yes \
  -o PubkeyAuthentication=yes \
  -o PreferredAuthentications=publickey \
  -o StrictHostKeyChecking=no \
  $STORAGE_ACCOUNT.$SFTP_USER_A@$STORAGE_ACCOUNT.blob.core.windows.net
```

Once connected, run these SFTP commands:

```
sftp> ls /             # Can see everything — THIS IS THE BUG
sftp> cd userB-home    # Can enter — THIS SHOULD FAIL
sftp> exit
```

---

## Step 9 — Apply ACL Fix

### Root Directory — Traverse Only (Cannot List)

```bash
az storage fs access set \
  --acl "user:$USER_A_OID:--x,user:$USER_B_OID:--x" \
  --path "/" \
  --file-system $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login
```

### User A Home — Full Access for A, None for B

```bash
az storage fs access set \
  --acl "user:$USER_A_OID:rwx,default:user:$USER_A_OID:rwx,user:$USER_B_OID:---,default:user:$USER_B_OID:---" \
  --path "/userA-home" \
  --file-system $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login
```

### User B Home — Full Access for B, None for A

```bash
az storage fs access set \
  --acl "user:$USER_B_OID:rwx,default:user:$USER_B_OID:rwx,user:$USER_A_OID:---,default:user:$USER_A_OID:---" \
  --path "/userB-home" \
  --file-system $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login
```

---

## Step 10 — Verify ACLs Applied Correctly

```bash
echo "=== ROOT ACL ==="
az storage fs access show \
  --path "/" \
  --file-system $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login \
  --query "acl"

echo "=== USER A HOME ACL ==="
az storage fs access show \
  --path "/userA-home" \
  --file-system $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login \
  --query "acl"

echo "=== USER B HOME ACL ==="
az storage fs access show \
  --path "/userB-home" \
  --file-system $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login \
  --query "acl"
```

---

## Step 11 — Validate the Fix via SFTP

### Test as User A

```bash
sftp \
  -i ~/.ssh/sftp_poc_key \
  -o IdentitiesOnly=yes \
  -o PubkeyAuthentication=yes \
  -o PreferredAuthentications=publickey \
  -o StrictHostKeyChecking=no \
  $STORAGE_ACCOUNT.$SFTP_USER_A@$STORAGE_ACCOUNT.blob.core.windows.net
```

| SFTP Command      | Expected Result          |
|-------------------|--------------------------|
| `ls /`            | ❌ Permission denied      |
| `cd userA-home`   | ✅ Success                |
| `ls userA-home`   | ✅ Lists contents         |
| `cd userB-home`   | ❌ Permission denied      |

### Test as User B

```bash
sftp \
  -i ~/.ssh/sftp_poc_key \
  -o IdentitiesOnly=yes \
  -o PubkeyAuthentication=yes \
  -o PreferredAuthentications=publickey \
  -o StrictHostKeyChecking=no \
  $STORAGE_ACCOUNT.$SFTP_USER_B@$STORAGE_ACCOUNT.blob.core.windows.net
```

| SFTP Command      | Expected Result          |
|-------------------|--------------------------|
| `ls /`            | ❌ Permission denied      |
| `cd userB-home`   | ✅ Success                |
| `ls userB-home`   | ✅ Lists contents         |
| `cd userA-home`   | ❌ Permission denied      |

---

## Step 12 — Test Default ACL Inheritance

```bash
# Create a subfolder inside userA-home
az storage fs directory create \
  --name "userA-home/new-subfolder" \
  --file-system $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login

# Verify ACLs inherited automatically
az storage fs access show \
  --path "/userA-home/new-subfolder" \
  --file-system $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login \
  --query "acl"

# Should show user:$USER_A_OID:rwx and user:$USER_B_OID:---
# without manually setting them — proving inheritance works
```

---

## ACL Permission Reference

| Permission | On Files | On Directories |
|------------|----------|----------------|
| `r` (Read) | Read file contents | List directory contents |
| `w` (Write) | Modify file | Create/delete in directory |
| `x` (Execute) | Run file | **Traverse/enter directory** |
| `---` | No access | No access |
| `--x` | — | Traverse only, cannot list |
| `rwx` | Full access | Full access |

---

## Directory Structure After Fix

```
/ (root)
├── ACL: user:sftpusera:--x    ← traverse only
├── ACL: user:sftpuserb:--x    ← traverse only
│
├── /userA-home/
│   ├── ACL: user:sftpusera:rwx   ← full access
│   └── ACL: user:sftpuserb:---   ← no access
│
└── /userB-home/
    ├── ACL: user:sftpusera:---   ← no access
    └── ACL: user:sftpuserb:rwx   ← full access
```

---

## Key Concepts

### Why HNS is Required
| Capability | Without HNS | With HNS |
|------------|-------------|----------|
| POSIX ACLs | ❌ | ✅ |
| Real directory permissions | ❌ | ✅ |
| Execute (X) bit on folders | ❌ | ✅ |
| SFTP support | ❌ | ✅ |

### Why Execute (X) on Root Matters
Without `--x` on the root directory, users can traverse the entire file system tree even if they cannot read files. The fix applies:
- `--x` on root and sibling directories → can traverse to reach home folder, cannot list
- `rwx` on home directory → full access within their own space
- `---` on other user directories → completely blocked

### Why SSH Key Auth (vs Password)
| Auth Method | Shared Key Required | Security |
|-------------|--------------------|---------:|
| Password | ✅ Yes | Medium |
| SSH Key | ❌ No | High |

SSH key authentication bypasses the `allowSharedKeyAccess` requirement, making it compatible with NIST SP 800-53 Rev. 5 and similar compliance policies.

---

## Troubleshooting

### Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `KeyBasedAuthenticationNotPermitted` | Shared key disabled | Add `--auth-mode login` to all storage commands |
| `FilesystemNotFound` | Container not created | Run Step 4 |
| `InvalidRequestPropertyValue` on resourceName | Path included in resource-name | Use container name only, not `container/path` |
| `allowSharedKeyAccess: false` after update | NIST/CIS policy at tenant level | Use SSH key auth instead of password |
| SSH disconnect after password sent | Shared key policy blocking auth | Switch to SSH key authentication |

### Useful Diagnostic Commands

```bash
# Check all storage account settings
az storage account show \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --query "{HNS:isHnsEnabled, SFTP:isSftpEnabled, LocalUser:isLocalUserEnabled, SharedKey:allowSharedKeyAccess}" \
  -o json

# Check local user state
az storage account local-user show \
  --account-name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --name $SFTP_USER_A \
  --query "{name:name, hasSshPassword:hasSshPassword, homeDirectory:homeDirectory, sid:sid}" \
  -o json

# Check blocking policies
az policy state list \
  --resource $(az storage account show \
    --name $STORAGE_ACCOUNT \
    --resource-group $RESOURCE_GROUP \
    --query "id" -o tsv) \
  --query "[?policyDefinitionAction=='deny'].{Policy:policyDefinitionName, Action:policyDefinitionAction}" \
  -o table
```

---

## Cleanup

```bash
az group delete \
  --name $RESOURCE_GROUP \
  --yes --no-wait

# Remove SSH key pair
rm ~/.ssh/sftp_poc_key
rm ~/.ssh/sftp_poc_key.pub
```

---

*Guide covers Azure SFTP + HNS + POSIX ACL isolation POC*
*Tested on macOS with Azure CLI 2.76.0+*
