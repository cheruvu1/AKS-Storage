# Azure SFTP POC — Full Setup Guide
### HNS(Hierarchical Namespace) + SFTP + ACL Isolation with SSH Key Authentication
 
### Prerequisites

bash

```jsx

az cloud set --name AzureUSGovernment
Az account list
az login 

 
## GOV
# Set all variables first
RESOURCE_GROUP="rg-sftp-poc"
LOCATION="USGovTexas"
STORAGE_ACCOUNT="stsftpprodbc001"         # must be globally unique, lowercase
CONTAINER="sftpcontainer"
SFTP_USER_A="sftpusera"
SFTP_USER_B="sftpuserb"


az account show --query "{SubscriptionID:id, Name:name}" -o json

```

---

### Step 1 — Create Resource Group

bash

```jsx
bash

az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION

# Verify
az group show --name $RESOURCE_GROUP --query "properties.provisioningState"
# Expected: "Succeeded"`
```

---

### Step 2 — Create Storage Account with HNS + SFTP Enabled

bash

```jsx
az storage account create \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --sku Standard_LRS \
  --kind StorageV2 \
  --enable-hierarchical-namespace true \
  --enable-sftp true \
  --allow-blob-public-access false \
  --min-tls-version TLS1_2

# Verify HNS and SFTP are both enabled
az storage account show \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --query "{HNS:isHnsEnabled, SFTP:isSftpEnabled}" \
  -o json

# Expected:
# {
#   "HNS": true,
#   "SFTP": true
# }
```

---

### Step 3 — Create the Blob Container

bash

```jsx
az storage fs directory create \
  --name "userA-home" \
  --file-system $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login
 

# Verify

az storage fs show \
  --name $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login \
  --query "name"
  
# Output

"sftpcontainer"

```

---

### Step 4 — Create Home Directories for Each User

bash

```jsx
**## Create userA-home directory** 

az storage fs directory create \
  --name "userA-home" \
  --file-system $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login
  

```

```jsx
**## Create userB-home directory** 

az storage fs directory create \
  --name "userB-home" \
  --file-system $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login


 
```

```jsx
**# Verify both exist**

az storage fs directory list \
  --file-system $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login \
  -o table
  
```

---

### Step 5 — Create SFTP Local Users

bash

**# Create User A - scoped to their home directory**


```jsx

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

```

**# Create User B - resource-name is CONTAINER only**
```jsx


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

---

### Step 6 — Generate and Save Passwords

bash

```jsx
**# Generate password for User A**
az storage account local-user regenerate-password \
  --account-name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --name $SFTP_USER_A \
  --query "sshPassword" \
  -o tsv
# SAVE THIS OUTPUT - shown only once  


**# Generate password for User B**
az storage account local-user regenerate-password \
  --account-name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --name $SFTP_USER_B \
  --query "sshPassword" \
  -o tsv

# SAVE THIS OUTPUT - shown only once  

```

---

### Step 7 — Get User Object IDs for ACL Assignment

bash

```jsx
**# Get the SID (used as OID for local SFTP users in ACLs)**

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

### Step 8 — Prove the Problem Exists (Before Fix)

bash

```jsx
**# Check root ACL - note permissive or absent user entries**

az storage fs access show \
  --path "/" \
  --file-system $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login
  
**# Output**

{
  "acl": "user::rwx,group::r-x,other::---",
  "group": "8d092d1e-c31c-470b-b589-b20da7a2d5c1",
  "owner": "8d092d1e-c31c-470b-b589-b20da7a2d5c1",
  "permissions": "rwxr-x---"
}

**# Now SFTP in as User A and try to browse into userB-home

 

sftp $STORAGE_ACCOUNT.$SFTP_USER_A@$STORAGE_ACCOUNT.blob.core.windows.net
# > ls /          (can see everything - this is the bug)
# > cd userB-home (should fail but currently succeeds)
```

---

### Step 9 — Apply the ACL Fix

bash

# Root: traverse only for both users - cannot list, cannot read

```jsx
az storage fs access set \
  --acl "user:$USER_A_OID:--x,user:$USER_B_OID:--x" \
  --path "/" \
  --file-system $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login
```

# User A home: full access for A, none for B

```jsx
az storage fs access set \
  --acl "user:$USER_A_OID:rwx,default:user:$USER_A_OID:rwx,user:$USER_B_OID:---,default:user:$USER_B_OID:---" \
  --path "/userA-home" \
  --file-system $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login

```
# User B home: full access for B, none for A

```jsx
az storage fs access set \
  --acl "user:$USER_B_OID:rwx,default:user:$USER_B_OID:rwx,user:$USER_A_OID:---,default:user:$USER_A_OID:---" \
  --path "/userB-home" \
  --file-system $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login

```
---

### Step 10 — Verify ACLs Applied Correctly

bash

```jsx
echo "=== ROOT ACL ===" 
az storage fs access show --path "/" \
  --file-system $CONTAINER --account-name $STORAGE_ACCOUNT \
  --auth-mode login --query "acl"

```

```jsx
echo "=== USER A HOME ACL ==="
az storage fs access show --path "/userA-home" \
  --file-system $CONTAINER --account-name $STORAGE_ACCOUNT \
  --auth-mode login --query "acl"

```
```jsx
echo "=== USER B HOME ACL ==="
az storage fs access show --path "/userB-home" \
  --file-system $CONTAINER --account-name $STORAGE_ACCOUNT \
  --auth-mode login --query "acl"`

```
---

### Step 11 — Validate the Fix via SFTP

bash 
# Test as User A

```jsx

sftp $STORAGE_ACCOUNT.$SFTP_USER_A@$STORAGE_ACCOUNT.blob.core.windows.net`

```

| Command in SFTP | Expected Result |
| --- | --- |
| `ls /` | ❌ Permission denied |
| `cd userA-home` | ✅ Success |
| `ls userA-home` | ✅ Lists contents |
| `cd userB-home` | ❌ Permission denied |


bash


# Test as User B

```jsx

sftp $STORAGE_ACCOUNT.$SFTP_USER_B@$STORAGE_ACCOUNT.blob.core.windows.net`

```

| Command in SFTP | Expected Result |
| --- | --- |
| `ls /` | ❌ Permission denied |
| `cd userB-home` | ✅ Success |
| `ls userB-home` | ✅ Lists contents |
| `cd userA-home` | ❌ Permission denied |

---

### Step 12 — Test Default ACL Inheritance

bash

### Create a subfolder inside userA-home

```jsx

az storage fs directory create \
  --name "userA-home/new-subfolder" \
  --file-system $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login
  
```

### Verify ACLs inherited automatically - no manual set needed

```jsx

az storage fs access show \
  --path "/userA-home/new-subfolder" \
  --file-system $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login \
  --query "acl"

```

### Should show user:$USER_A_OID:rwx and user:$USER_B_OID:---
### without you having manually set them - proving inheritance works`

---

### Cleanup (After POC)

bash

```jsx

az group delete \
  --name $RESOURCE_GROUP \
  --yes --no-wait`

```



*Guide covers Azure SFTP + HNS + POSIX ACL isolation POC*
*Tested on macOS with Azure CLI 2.76.0+*
