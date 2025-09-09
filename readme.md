# Hoppscotch Deployment on AKS with Cloudflare Tunnel

This guide documents the complete deployment of Hoppscotch (API testing tool) on Azure Kubernetes Service (AKS) using Cloudflare Tunnel for secure external access. We are also applying Zero-Trust principles by ensuring that all images are imported into our private ACR before being pulled or used by the Pods, and also creating a private endpoint for the database to be accessed privately. 

## Architecture Overview

- **Application**: Hoppscotch AIO (All-In-One) container v2025.8.1
- **Database**: Azure PostgreSQL Flexible Server
- **Ingress**: Cloudflare Tunnel (managed tunnel with token)
- **Platform**: Azure Kubernetes Service (AKS)
- **Image Registry**: Azure Container Registry (ACR)

## Installation Sequence with dependencies

```mermaid
flowchart TD
    A[Set Environment Variables] --> B[Create Azure PostgreSQL Database]
    A --> D[Import Images to ACR]
    A --> F[Configure Cloudflare Tunnel]
    
    B --> C[Create Private Endpoint]
    B --> E[Create PostgreSQL Extension]
    B --> H[Create Kubernetes Secrets]
    
    D --> G[Create Kubernetes Namespace]
    F --> G
    
    G --> I[Create ConfigMap]
    
    H --> J[Deploy Cloudflared]
    I --> K[Deploy Hoppscotch Application]
    H --> K
    
    J --> L[Create ClusterIP Service]
    K --> L
    
    L --> M[Run Database Migration]
    M --> N[Verify Application Access]
    N --> O[Configure Admin Access]
    
    style A fill:#e1f5fe,color:#000000
    style B fill:#f3e5f5,color:#000000
    style C fill:#f3e5f5,color:#000000
    style D fill:#fff3e0,color:#000000
    style E fill:#f3e5f5,color:#000000
    style F fill:#e8f5e8,color:#000000
    style G fill:#fff8e1,color:#000000
    style H fill:#fff8e1,color:#000000
    style I fill:#fff8e1,color:#000000
    style J fill:#fff8e1,color:#000000
    style K fill:#fff8e1,color:#000000
    style L fill:#fff8e1,color:#000000
    style M fill:#fff8e1,color:#000000
    style N fill:#e8f5e8,color:#000000
    style O fill:#e8f5e8,color:#000000
```

## Prerequisites

- Azure CLI installed and configured
- kubectl configured for your AKS cluster
- Cloudflare account with a domain
- Azure PostgreSQL Flexible Server
- Azure Container Registry (ACR) integrated with AKS
- AKS cluster with node taints configured for your environment (e.g., `platform=lin01n:NoSchedule` for nprod, `platform=lin01:NoSchedule` for prod)

## Directory Structure

```
hoppscotch/
├── 1-namespace.yaml
├── 2-secret.yaml
├── 3-cloudflared.yaml
├── 4-deployment.yaml
├── 5-cluster-ip-service.yaml
├── 6-config-map.yaml
├── 7-db-migration-job.yaml 
└── 8-postgres-client.yaml
```

## Environment Variables

Before starting, set the following environment variables according to your target environment:

### Variables needed
```bash
export REGION="YOUR_REGION"
export ENV="YOUR_ENV"
export RESOURCE_GROUP="YOUR_DB_RG"
export ACR_NAME="YOUR_ACR"
export NODE_TAINT_VALUE="YOUR_NODE_NAME"
export DOMAIN_SUFFIX="YOUR_DOMAIN"
```

### Example
```bash
export ENV="prod"
export RESOURCE_GROUP="rg-$REGION-db-pg-$ENV"
export ACR_NAME="acr$ENV"
export NODE_TAINT_VALUE="lin01"
export DOMAIN_SUFFIX="nexops.dev"
```

### Common Variables
```bash
export POSTGRES_SERVER_NAME="pg-$$REGION-${ENV}-hoppscotch-01"
export POSTGRES_DB_NAME="hoppscotch"
export POSTGRES_USER="admin" # Change this!
export POSTGRES_PASSWORD="YourSecurePassword#123!"  # Change this!
export TUNNEL_NAME="tunnel-hoppscotch-${ENV}"
export APP_HOSTNAME="hopp.${DOMAIN_SUFFIX}"
```

## Step 1: Create Azure PostgreSQL Database

```bash
# Create PostgreSQL Flexible Server
az postgres flexible-server create \
  --resource-group $RESOURCE_GROUP \
  --name $POSTGRES_SERVER_NAME \
  --location $REGION \
  --admin-user $POSTGRES_USER \
  --admin-password $POSTGRES_PASSWORD \
  --sku-name Standard_B1ms \
  --tier Burstable \
  --public-access 0.0.0.0 \
  --storage-size 32 \
  --version 14

# Create a Private Endpoint for the PostgreSQL server
# Make sure you have a virtual network and subnet ready (replace values as needed)
az network private-endpoint create \
  --name "pe-${POSTGRES_SERVER_NAME}" \
  --resource-group $RESOURCE_GROUP \
  --vnet-name "YOUR_VNET_NAME" \
  --subnet "YOUR_SUBNET_NAME" \
  --private-connection-resource-id $(az postgres flexible-server show --resource-group $RESOURCE_GROUP --name $POSTGRES_SERVER_NAME --query id -o tsv) \
  --group-id "postgresqlServer" \
  --connection-name "pe-conn-${POSTGRES_SERVER_NAME}"

# Approve the private endpoint connection if required
# az network private-endpoint-connection approve \
#   --resource-group $RESOURCE_GROUP \
#   --name <connection-name> \
#   --server-name $POSTGRES_SERVER_NAME

# Create the database
az postgres flexible-server db create \
  --resource-group $RESOURCE_GROUP \
  --server-name $POSTGRES_SERVER_NAME \
  --database-name $POSTGRES_DB_NAME

# Configure firewall (allow Azure services)
az postgres flexible-server firewall-rule create \
  --resource-group $RESOURCE_GROUP \
  --name $POSTGRES_SERVER_NAME \
  --rule-name "AllowAzureServices" \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0

# Enable pg_trgm extension
az postgres flexible-server parameter set \
  --resource-group $RESOURCE_GROUP \
  --server-name $POSTGRES_SERVER_NAME \
  --name azure.extensions \
  --value "pg_trgm,pg_stat_statements,pgcrypto"

# Enable the pg_trgm extension on your Azure PostgreSQL server
az postgres flexible-server parameter set \
  --resource-group $RESOURCE_GROUP \
  --server-name $POSTGRES_SERVER_NAME \
  --name shared_preload_libraries \
  --value "pg_trgm,pg_stat_statements"

# See step 3 if you're getting an error for pg_trgm

# Restart server to apply extension changes
az postgres flexible-server restart \
  --resource-group $RESOURCE_GROUP \
  --name $POSTGRES_SERVER_NAME
```

## Step 2: Import Images to private ACR
We want to make sure that images are stored and pulled privately.

Use this script https://github.com/nocticdr/azure-container-registry/blob/main/import_image_github_acr.sh or follow the steps below:

```bash
# Import Hoppscotch image (specific version used)
az acr import \
  --name $ACR_NAME \
  --source docker.io/hoppscotch/hoppscotch:2025.8.1 \
  --image hoppscotch:v2025.8.1

# Import Cloudflared image
az acr import \
  --name $ACR_NAME \
  --source docker.io/cloudflare/cloudflared:latest \
  --image cloudflared:latest

# Import PostgreSQL client image for database troubleshooting and operations
az acr import \
  --name $ACR_NAME \
  --source docker.io/postgres:15-alpine \
  --image postgres:15-alpine
```

## Step 3: Create PostgreSQL Extension

Connect to your database and create the required extension:

### Using PostgreSQL Client Pod (8-postgres-client.yaml)
```bash
kubectl apply -f 8-postgres-client.yaml

# Then connect to database
kubectl exec -it postgres-client -n hoppscotch -- psql -h ${POSTGRES_SERVER_NAME}.postgres.database.azure.com -U ${POSTGRES_USER} -d ${POSTGRES_DB_NAME} -p 5432
```

```sql
-- Create the pg_trgm extension
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Verify extension was created
\dx

-- Exit
\q
```

## Step 4: Configure Cloudflare Tunnel

### Create Managed Tunnel

1. Go to Cloudflare Dashboard → Zero Trust → Access → Tunnels
2. Create a new tunnel named `${TUNNEL_NAME}` (e.g., `tunnel-hoppscotch-prod`)
3. Install and run connector (choose Cloudflare managed)
4. Copy the tunnel token for use in Kubernetes secret
5. Configure public hostnames:
   - `${APP_HOSTNAME}` (e.g., `hopp.nexops.dev`) → `http://hoppscotch-aio-service.hoppscotch.svc.cluster.local:80`

## Step 5: Create Kubernetes Resources

### 1. Create Namespace 
Replace placeholders and apply

```bash
kubectl apply -f 1-namespace.yaml
```

### 2. Create Secrets
Replace YOUR_TUNNEL_TOKEN_FROM_CLOUDFLARE, YOUR_PASSWORD,POSTGRES_USER, POSTGRES_SERVER_NAME and YOUR_DATA_ENCRYPTION_KEY, and apply

> **⚠️ Important Note:**  If your DB Password contains special characters, you need to encode them. However, when connecting from postgres client later, you need to use the raw/plain password.

```bash
kubectl apply -f 2-secret.yaml
```

### 3. Cloudflared Deployment
Replace UPDATE YOUR_ACR_NAME and YOUR_NODE_NAME and apply

```bash
kubectl apply -f 3-cloudflared.yaml
```

### 4. Create ConfigMap 
Replace DOMAIN_SUFFIX and apply

```bash
kubectl apply -f 4-config-map.yaml
```

### 5. Hoppscotch Deployment
Replace YOUR_ACR_NAME and YOUR_NODE_NAME and apply

```bash
kubectl apply -f 5-deployment.yaml
```

### 6. Create ClusterIP Service 

```bash
kubectl apply -f 6-cluster-ip-service.yaml
```

### 7. Migration Job (7-db-migration-job.yaml)
Replace YOUR_NODE_NAME and YOUR_ACR_NAME and apply

```bash
kubectl apply -f 7-db-migration-job.yaml
```

## Step 6: Deploy Application

Apply the Kubernetes manifests in order:

```bash
kubectl apply -f 1-namespace.yaml
kubectl apply -f 2-secret.yaml
kubectl apply -f 3-cloudflared.yaml
kubectl apply -f 4-config-map.yaml
kubectl apply -f 5-deployment.yaml
kubectl apply -f 6-cluster-ip-service.yaml
```
Check and make sure there are no errors with the above. The hoppscotch deployment could still result in errors due to incomplete DB migration. We fix this below.

## Step 7: Run Database Migration

```bash
# Run initial database migration
kubectl apply -f 7-db-migration-job.yaml

# Monitor migration
kubectl logs -n hoppscotch job/hoppscotch-db-migration -f

# Restart hoppscotch containers
kubectl rollout restart deployment/hoppscotch-aio -n hoppscotch 
```

## Step 8: Check if application is running on the browser
Access the domain from the browser to confirm everything is ok. 

You may also check the /backend/health endpoint to confirm there are no errors.


## Step 8: Configure Admin Access - Only if not auto-created

### Database Update (Direct)
Replace YOUR_NODE_NAME and YOUR_ACR_NAME, POSTGRES_SERVER_NAME and POSTGRES_USER

```bash
# Create postgres client pod

kubectl apply -f 8-postgres-client.yaml
kubectl exec -it postgres-client -n hoppscotch -- psql -h ${POSTGRES_SERVER_NAME}.postgres.database.azure.com -U ${POSTGRES_USER} -d ${POSTGRES_DB_NAME} -p 5432

# Make user admin (replace with your email)
UPDATE "User" SET "isAdmin" = true WHERE "email" = 'your-email@domain.com';

# Verify
SELECT "uid", "displayName", "email", "isAdmin" FROM "User" WHERE "email" = 'your-email@domain.com';

\q
```

## Key Configuration Notes

### Critical Settings

1. **Encryption Key**: Must be exactly 32 characters
   ```bash
   openssl rand -hex 16  # Generates exactly 32 hex characters
   ```

2. **URL Encoding**: Special characters in passwords must be URL-encoded
   - `#` becomes `%23`
   - `@` becomes `%40`
   - `+` becomes `%2B`

3. **Subpath Access**: With `ENABLE_SUBPATH_BASED_ACCESS: "true"`:
   - Main app: `/`
   - Admin: `/admin`
   - Backend: `/backend/*`
   - All served on port 3000, routed via Cloudflare to port 80

4. **Node Tolerations**: Required for AKS clusters with node taints (adjust for your environment):
   ```yaml
   tolerations:
     - key: "platform"
       operator: "Equal"
       value: "lin01"  
       effect: "NoSchedule"
   ```

5. **Resource Limits**: Mandatory if you have an Azure Policy for it:
   ```yaml
   resources:
     requests:
       memory: "512Mi"
       cpu: "200m"
     limits:
       memory: "1Gi"
       cpu: "500m"
   ```

### Cloudflare Tunnel Configuration

Using managed tunnels with token authentication:
- No config files needed in Kubernetes
- Configure public hostnames in Cloudflare dashboard
- Token stored in Kubernetes secret

## Troubleshooting

### Common Issues

1. **Crypto Key Length Error**
   ```
   RangeError: Invalid key length
   ```
   - Solution: Ensure `DATA_ENCRYPTION_KEY` is exactly 32 characters
   - Generate with: `openssl rand -hex 16`

2. **Database Connection Issues**
   ```
   error connecting to postgres
   ```
   - Verify URL encoding of special characters in password
   - Check Azure PostgreSQL firewall rules
   - Ensure `pg_trgm` extension is enabled

3. **Extension Errors**
   ```
   extension "pg_trgm" is not allow-listed
   ```
   - Enable extension in Azure PostgreSQL parameters
   - Restart PostgreSQL server
   - Create extension manually in database

4. **Admin 404 Errors**
   - Verify Cloudflare tunnel routes `/admin*` to port 80
   - Check `ENABLE_SUBPATH_BASED_ACCESS` is `"true"`
   - Ensure service exposes port 80

5. **Pod Scheduling Issues**
   ```
   pod didn't trigger scale-up: it didn't match node selector
   ```
   - Verify node tolerations match cluster taint configuration
   - Check if nodes are labeled correctly

### Health Checks

```bash
# Check application health (replace with your actual domain)
curl https://hopp.${DOMAIN_SUFFIX}/backend/health

# Should return:
# {"status":"ok","info":{"database":{"status":"up"}},"error":{},"details":{"database":{"status":"up"}}}

# Check pods
kubectl get pods -n hoppscotch

# Check logs
kubectl logs -n hoppscotch deployment/hoppscotch-aio -f

# Check tunnel connectivity
kubectl logs -n hoppscotch deployment/cloudflared
```

## Access URLs

- **Main Application**: https://hopp.${DOMAIN_SUFFIX} (e.g., https://hopp.nexops.dev)
- **Admin Dashboard**: https://hopp.${DOMAIN_SUFFIX}/admin
- **GraphQL Endpoint**: https://hopp.${DOMAIN_SUFFIX}/backend/graphql
- **API Health**: https://hopp.${DOMAIN_SUFFIX}/backend/health
- **API v1**: https://hopp.${DOMAIN_SUFFIX}/backend/v1

## Security Considerations

- Database credentials stored in Kubernetes secrets
- Cloudflare Tunnel eliminates need for public load balancers
- All traffic encrypted via HTTPS/WSS
- Admin access controlled via database user flags
- Resource limits enforced via Azure Policy
- Node isolation via taints and tolerations

## Monitoring

```bash
# Monitor all resources
kubectl get all -n hoppscotch

kubectl logs -n hoppscotch deployment/hoppscotch-aio -f

# Watch pod status
kubectl get pods -n hoppscotch -w

# Check events
kubectl get events -n hoppscotch --sort-by='.lastTimestamp'

# Check resource usage
kubectl top pods -n hoppscotch
```

## Version Information

- **Hoppscotch**: v2025.8.1
- **PostgreSQL**: 14
- **Cloudflared**: latest
- **Node.js**: v22.16.0 (in container)
- **Kubernetes**: Compatible with AKS


# Hopscotch OAuth & SMTP Configuration

This guide will help you configure OAuth authentication and SMTP email functionality for your Hopscotch instance.

## Prerequisites

- Hopscotch instance deployed and accessible
- Microsoft Azure AD app registration (for OAuth)
- SendGrid account with API key

---

## OAuth Configuration (Microsoft)

### Step 1: Create Azure AD App Registration

1. Go to [Azure Portal](https://portal.azure.com/)
2. Navigate to **Azure Active Directory** > **App registrations**
3. Click **New registration**
4. Fill in the application details:
   - **Name**: Your app name (e.g., "Hopscotch")
   - **Supported account types**: Choose based on your needs
   - **Redirect URI**: Leave blank for now (will be generated by Hopscotch)

### Step 2: Get Client Credentials

1. After creating the app, go to **Overview** tab
2. Copy the **Application (client) ID**
3. Go to **Certificates & secrets** tab
4. Click **New client secret**
5. Add a description and set expiration
6. Copy the **Value** (this is your Client Secret)

### Step 3: Configure OAuth in Hopscotch

1. Navigate to **Settings** > **Authentication** in your Hopscotch instance
2. Click on the **OAuth** tab
3. Enable **Microsoft** provider
4. Fill in the configuration:
   - **Client Id**: Paste your Application (client) ID
   - **Client Secret**: Paste your client secret value
   - **Tenant**: Your Azure AD tenant ID (found in Azure AD Overview)

### Step 4: Update Azure AD Redirect URI

1. After saving the OAuth configuration in Hopscotch, note the **Callback URL** that's automatically generated
2. Go back to your Azure AD app registration
3. Navigate to **Authentication** tab
4. Click **Add a platform** > **Web**
5. Add the callback URL from Hopscotch (typically: `https://your-domain.com/backend/v1/auth/microsoft/callback`)

### Step 5: Configure API Permissions

1. In your Azure AD app, go to **API permissions**
2. Click **Add a permission** > **Microsoft Graph** > **Delegated permissions**
3. Add the following permissions:
   - `email`
   - `offline_access`
   - `openid`
   - `User.Read`
4. Click **Grant admin consent** (if you have admin privileges)

> **Note**: The scope `email offline_access openid User.Read` will be automatically configured by Hopscotch for SSO functionality.

---

## SMTP Configuration (SendGrid)

### Step 1: Create SendGrid API Key

1. Log into your [SendGrid Dashboard](https://app.sendgrid.com/)
2. Navigate to **Settings** > **API Keys**
3. Click **Create API Key**
4. Choose **Restricted Access** and configure permissions:
   - **Mail Send**: Full Access
   - **Mail Settings**: Full Access
5. Name your API key (e.g., "Hopscotch SMTP")
6. Click **Create & View**
7. **Copy and save the API key** (you won't be able to see it again)

### Step 2: Verify Sender Email

1. In SendGrid, go to **Settings** > **Sender Authentication**
2. Choose one of the following options:
   - **Single Sender Verification**: Verify `hoppscotch@qndev.net`
   - **Domain Authentication**: Authenticate your entire domain `qndev.net`
3. Follow the verification process (email verification or DNS records)

### Step 3: Configure SMTP in Hopscotch

1. Navigate to **Settings** > **SMTP** in your Hopscotch instance
2. Enable **SMTP** configurations
3. Check **Use Custom SMTP Configurations**
4. Fill in the configuration:

   **Mailer Smtp Url:**
   ```
   smtp://apikey:YOUR_SENDGRID_API_KEY@smtp.sendgrid.net:587
   ```
   
   **Mailer From Address:**
   ```
   hoppscotch@qndev.net
   ```

> **Important**: Replace `YOUR_SENDGRID_API_KEY` with the actual API key you created in Step 1.

---

## Testing the Configuration

### Test OAuth
1. Log out of Hopscotch if currently logged in
2. Try to log in using the Microsoft OAuth option
3. You should be redirected to Microsoft login and then back to Hopscotch

### Test SMTP
1. Trigger an email functionality (like password reset or user invitation)
2. Check that emails are being sent successfully
3. Monitor SendGrid dashboard for email delivery statistics

---

## Troubleshooting

### OAuth Issues
- **Redirect URI mismatch**: Ensure the callback URL in Azure matches exactly what Hopscotch generates
- **Permission denied**: Make sure all required API permissions are granted and admin consent is provided
- **Tenant issues**: Verify the correct tenant ID is used

### SMTP Issues
- **Authentication failed**: Double-check your SendGrid API key and ensure it has the correct permissions
- **Sender not verified**: Ensure `hoppscotch@qndev.net` is verified in SendGrid
- **Port issues**: Try port 25, 465, or 2525 if 587 doesn't work

---

## Security Notes

- Store API keys and client secrets securely
- Regularly rotate SendGrid API keys
- Monitor Azure AD sign-in logs for suspicious activity
- Use environment variables for sensitive configuration in production

This deployment has been tested and verified working with the specific configuration above.