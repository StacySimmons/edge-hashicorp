# HashiCorp Vault to Cribl Edge Integration - Complete Deployment Guide

This guide covers the complete setup for sending both Vault audit logs and telemetry metrics to Cribl Edge in a Kubernetes environment.

## Prerequisites

- Kubernetes cluster with kubectl access
- Helm 3.x installed
- HashiCorp Vault Helm chart repository added
- Cribl Edge already deployed in Kubernetes

## Overview

**Data Flows:**
- **Audit Logs**: Vault → Cribl Edge (Port 9514, TCP)
- **Telemetry**: Vault → Cribl Edge (Port 8125, UDP, StatsD format)

## Part 1: Cribl Edge Configuration

### Step 1: Update Edge Helm Values

Create or update your Cribl Edge `values.yaml` to include the service and port configuration:

```yaml
# edge-values.yaml
# Your existing Edge configuration...

# Service configuration for Vault integration
service:
  # Enable additional service for Vault integration
  vault:
    enabled: true
    name: cribl-edge-service
    type: ClusterIP
    ports:
      - name: syslog
        port: 9514
        targetPort: 9514
        protocol: TCP
      - name: statsd
        port: 8125
        targetPort: 8125
        protocol: UDP

# Pod configuration - expose required ports
podSpec:
  containers:
    edge:
      ports:
        - name: syslog
          containerPort: 9514
          protocol: TCP
        - name: statsd
          containerPort: 8125
          protocol: UDP

# If your Edge chart doesn't support the above structure,
# use the standard service configuration:
service:
  enabled: true
  type: ClusterIP
  ports:
    - name: syslog
      port: 9514
      targetPort: 9514
      protocol: TCP
    - name: statsd
      port: 8125
      targetPort: 8125
      protocol: UDP
```

### Step 2: Deploy/Update Cribl Edge

Apply the updated configuration:

```bash
# Update your Edge deployment with the new values
helm upgrade --install cribl-edge <CRIBL_EDGE_CHART> \
  -n cribl \
  --create-namespace \
  -f edge-values.yaml
```

**Note:** Replace `<CRIBL_EDGE_CHART>` with your actual Edge Helm chart reference (e.g., `cribl/edge` or path to local chart).

### Step 3: Configure Edge Sources

#### Syslog Source (for Audit Logs)

In your Cribl Edge configuration:

1. **Create Syslog Source:**
   - Source Type: `Syslog`
   - Listen Address: `0.0.0.0`
   - Port: `9514`
   - Protocol: `TCP`
   - Source ID: `vault_audit_logs`

2. **Fields to add:**
   - `source_type`: `vault:audit`
   - `index`: `vault_audit` (or your preferred index)

#### StatsD Source (for Telemetry)

1. **Create StatsD Source:**
   - Source Type: `StatsD`
   - Listen Address: `0.0.0.0`
   - Port: `8125`
   - Protocol: `UDP`
   - Source ID: `vault_metrics`

2. **Fields to add:**
   - `source_type`: `vault:telemetry`
   - `index`: `vault_metrics` (or your preferred index)

### Step 4: Verify Service Creation

After the Helm upgrade, verify the service was created:

```bash
# Check that the service exists
kubectl get svc -n cribl cribl-edge-service

# Verify service endpoints
kubectl get endpoints -n cribl cribl-edge-service

# Test service DNS resolution
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup cribl-edge-service.cribl.svc.cluster.local
```

## Part 2: HashiCorp Vault Configuration

### Step 1: Prepare Vault Helm Values

Create or update your `vault-values.yaml`:

```yaml
# vault-values.yaml
global:
  enabled: true

server:
  enabled: true
  
  image:
    repository: "hashicorp/vault"
    tag: "1.20.2"
    pullPolicy: IfNotPresent
  
  resources:
    requests:
      memory: 256Mi
      cpu: 250m
    limits:
      memory: 1Gi
      cpu: 500m
  
  standalone:
    enabled: true
    config: |
      ui = true
      
      listener "tcp" {
        tls_disable = 1
        address = "[::]:8200"
        cluster_address = "[::]:8201"
      }
      
      storage "file" {
        path = "/vault/data"
      }
      
      # Telemetry configuration - sends metrics to Cribl Edge
      telemetry {
        statsd_address = "cribl-edge-service.cribl.svc.cluster.local:8125"
        disable_hostname = true
        enable_hostname_label = true
      }
      
      disable_mlock = true
  
  ingress:
    enabled: false

# Disable injector
injector:
  enabled: false

# Enable UI
ui:
  enabled: true
  serviceType: ClusterIP
```

### Step 2: Deploy/Update Vault

```bash
# Add HashiCorp Helm repo (if not already added)
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

# Deploy or upgrade Vault
helm upgrade --install vault hashicorp/vault \
  -n vault \
  --create-namespace \
  -f vault-values.yaml
```

### Step 3: Initialize and Unseal Vault

**If Vault is new/uninitialized:**

```bash
# Initialize Vault
kubectl exec -n vault vault-0 -- vault operator init

# CRITICAL: Save the unseal keys and root token from the output!
```

**Unseal Vault (required every time Vault restarts):**

```bash
# Use any 3 of the 5 unseal keys from initialization
kubectl exec -n vault vault-0 -- vault operator unseal <UNSEAL_KEY_1>
kubectl exec -n vault vault-0 -- vault operator unseal <UNSEAL_KEY_2>
kubectl exec -n vault vault-0 -- vault operator unseal <UNSEAL_KEY_3>
```

**Login to Vault:**

```bash
# Use the root token from initialization
kubectl exec -n vault vault-0 -- vault login <ROOT_TOKEN>
```

### Step 4: Enable Audit Logging

Choose **ONE** of the following audit device configurations:

#### Option A: Socket Audit Device (Recommended)

```bash
kubectl exec -n vault vault-0 -- vault audit enable socket \
  address=cribl-edge-service.cribl.svc.cluster.local:9514 \
  socket_type=tcp
```

#### Option B: Syslog Audit Device (Alternative)

```bash
kubectl exec -n vault vault-0 -- vault audit enable syslog \
  facility=local0 \
  tag=vault \
  address=cribl-edge-service.cribl.svc.cluster.local:9514
```

## Part 3: Testing and Verification

### Test Audit Logs

Generate some audit events:

```bash
# Enable KV secrets engine
kubectl exec -n vault vault-0 -- vault secrets enable -path=secret kv

# Create a secret (generates audit log)
kubectl exec -n vault vault-0 -- vault kv put secret/test username=testuser password=testpass

# Read the secret (generates audit log)
kubectl exec -n vault vault-0 -- vault kv get secret/test

# Failed authentication attempt (generates audit log)
kubectl exec -n vault vault-0 -- vault login token=invalid-token-123
```

### Test Telemetry

Telemetry flows automatically once Vault is unsealed. Check for metrics in your Edge StatsD source.

### Verify Configuration

```bash
# Check audit devices are enabled
kubectl exec -n vault vault-0 -- vault audit list

# Check Vault status
kubectl exec -n vault vault-0 -- vault status

# Check Edge service is accessible from Vault namespace
kubectl exec -n vault vault-0 -- nslookup cribl-edge-service.cribl.svc.cluster.local
```

## Important Notes

### Vault Sealing Behavior

- **Vault seals automatically** after restarts for security
- **Audit logging stops** when Vault is sealed
- **Manual unsealing required** after each restart unless auto-unseal is configured
- **Telemetry also stops** when sealed

### Service Discovery

- Uses Kubernetes DNS: `cribl-edge-service.cribl.svc.cluster.local`
- Adjust namespace names if different from `cribl` and `vault`
- Ensure network policies allow communication between namespaces

### Port Requirements

- **TCP 9514**: Audit logs (syslog format)
- **UDP 8125**: Telemetry metrics (StatsD format)
- Both ports must be open and accessible from Vault namespace

### Troubleshooting

**No audit logs flowing:**
1. Verify Vault is unsealed: `kubectl exec -n vault vault-0 -- vault status`
2. Check audit devices: `kubectl exec -n vault vault-0 -- vault audit list`
3. Test network connectivity from Vault to Edge service
4. Check Edge syslog source configuration and logs

**No telemetry flowing:**
1. Verify Vault is unsealed
2. Check Edge StatsD source configuration
3. Test UDP connectivity on port 8125
4. Check Vault logs for StatsD connection errors

**Service not found errors:**
1. Verify service name and namespace
2. Check service selector matches Edge pod labels
3. Ensure DNS resolution works between namespaces

## Auto-Unseal Configuration (Optional)

For production environments, consider configuring auto-unseal to eliminate manual unsealing:

- **AWS KMS**: Use AWS encryption keys
- **Azure Key Vault**: Use Azure encryption keys  
- **Google Cloud KMS**: Use GCP encryption keys
- **Transit Seal**: Use another Vault instance

Auto-unseal ensures audit logging resumes automatically after Vault restarts.

## Security Considerations

- Store unseal keys and root tokens securely
- Use least-privilege authentication for production
- Consider enabling TLS for production deployments
- Implement proper RBAC for Vault access
- Monitor audit logs for security events

This configuration provides both audit logging and telemetry from Vault to Cribl Edge with automatic service discovery and proper port exposure.