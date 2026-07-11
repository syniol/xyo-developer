# 🐳 XYO Enrichment Docker Deployment

This guide outlines how to deploy the XYO Enrichment Service infrastructure locally or on-premise using Docker Compose.

---

## Prerequisites

Before starting, ensure your host environment meets the following requirements:

1. **Docker Engine**: Version `20.10.0` or higher.
2. **Docker Compose**: Version `2.0.0` or higher.
3. **Hardware**: 
   - CPU: 2+ cores (4+ recommended for high-throughput).
   - RAM: 4GB minimum (8GB recommended).
   - Disk: SSD or NVMe storage volume mounted at `/var/lib/xyo/logos` (used for caching merchant logo files).
4. **Registry Credentials**: Credentials provided by Syniol Limited to access the private container registry (`cr.syniol.com`).
5. **License Key**: A valid `XYO_LICENSE_KEY` for service activation.

---

## Deployment Steps

Follow these steps to spin up the entire stack:

### Step 1: Log in to the Syniol Container Registry
Authenticate with Syniol's private registry using your client credentials:
```bash
docker login cr.syniol.com
```
*Enter your username and password when prompted.*

### Step 2: Prepare Logo Cache Directory
The stack maps a local host directory to store and serve merchant logos. Create this directory and ensure the Docker containers have permissions to write to it:
```bash
sudo mkdir -p /var/lib/xyo/logos
sudo chmod 777 /var/lib/xyo/logos
```

### Step 3: Configure Environment Variables
Set your Syniol license key in your environment. This key is used by the containers on startup for activation:
```bash
export XYO_LICENSE_KEY="your-activation-license-key-here"
```

### Step 4: Launch the Stack
Start the services in detached mode:
```bash
docker compose up -d
```

### Step 5: Verify Services are Running
Check the status of the containers:
```bash
docker compose ps
```

You should see all 5 containers in the `running` state (and `postgres` reporting as `healthy`).

To view logs for any of the containers:
```bash
docker compose logs -f xyo-gateway
```

---

## Verification & Testing

Once the services are up, test that the API is responding correctly using `curl`:

```bash
curl -X POST http://localhost:8080/v1/enrich \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your-api-key-if-configured" \
  -d '{
    "description": "NETFLIX.COM GBR",
    "countryCode": "GB"
  }'
```

### Expected Response
```json
{
  "status": "success",
  "data": {
    "merchant": {
      "name": "Netflix",
      "domain": "netflix.com",
      "logoUrl": "http://localhost:8080/logos/netflix.png",
      "description": "Netflix is a subscription-based streaming service."
    },
    "category": {
      "name": "Entertainment",
      "mcc": "4899",
      "confidence": 0.99
    }
  }
}
```

---

## Clean Up

To stop and remove all containers, networks, and shared storage definitions created by this stack:

```bash
docker compose down
```

> [!WARNING]
> Running `docker compose down -v` will delete the persistent PostgreSQL volume containing cached API responses.
