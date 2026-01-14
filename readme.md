# Camel K Demo - Command Reference

This README contains all the commands needed to set up and run Camel K integrations on Kubernetes.


## Prerequisites

- Kubernetes cluster running (e.g., Docker Desktop with Kubernetes enabled)
- kubectl installed and configured
- **Helm installed** ([Helm install guide](https://helm.sh/docs/intro/install/))
- Local Docker registry running (optional, for custom builds)


## Setup Local Docker Registry (Optional)

```bash
docker run -d -p 5000:5000 --restart=always --name registry registry:2
```
## Remove Local Docker Registry

To stop and remove the local Docker registry container:

```bash
docker stop registry && docker rm registry
```

### Important: Configure Docker Desktop for Insecure Registry

If you are using Docker Desktop and your registry is not using HTTPS, you must add the following to your Docker Desktop settings:

1. Open Docker Desktop preferences.
2. Go to **Settings > Docker Engine**.
3. Add or update the `insecure-registries` section to include:

```json
  "insecure-registries": [
    "host.docker.internal:5000"
  ]
```

4. Click **Apply & Restart** to save changes.

This allows your local Kubernetes and Docker to push/pull images from the local registry without TLS.

Check registry contents:
```bash
curl http://localhost:5000/v2/_catalog
```


## 1. Install Camel K Operator with Helm

You need Helm installed to deploy the Camel K operator. Example commands:

```bash
kubectl create ns camel-k
helm repo add camel-k https://apache.github.io/camel-k/charts/
helm install camel-k camel-k/camel-k -n camel-k
```

## 2. Deploy IntegrationPlatform

The IntegrationPlatform configures how Camel K builds and deploys integrations.

```bash
kubectl apply -f itp.yaml -n camel-k
```

Check IntegrationPlatform status:
```bash
kubectl get integrationplatform -n camel-k
kubectl describe integrationplatform camel-k -n camel-k
```

## 2. Deploy YAML-based Integration

Deploy a simple timer-based integration using YAML DSL:

```bash
kubectl apply -f helloworld.yaml -n camel-k
```

## 3. Deploy Java-based Integration (Inline)

For Java-based integrations, use inline content in the Integration YAML:

```bash
kubectl apply -f helloworld-inline.yaml -n camel-k
```

**Note:** The inline approach embeds Java source code directly in the Integration YAML. This avoids ConfigMap mounting issues.

## Checking Integration Status

### List all integrations
```bash
kubectl get integration -n camel-k
```

### Get detailed integration information
```bash
kubectl describe integration <integration-name> -n camel-k
```

Example:
```bash
kubectl describe integration helloworld -n camel-k
```

### Check integration pods
```bash
kubectl get pods -n camel-k
```

### Check specific integration pod
```bash
kubectl get pods -l camel.apache.org/integration=<integration-name> -n camel-k
```

Example:
```bash
kubectl get pods -l camel.apache.org/integration=helloworld -n camel-k
```

## Viewing Logs

### Get logs from integration pod
```bash
kubectl logs -l camel.apache.org/integration=<integration-name> -n camel-k
```

Example:
```bash
kubectl logs -l camel.apache.org/integration=helloworld -n camel-k
```

### Get last 50 lines of logs
```bash
kubectl logs -l camel.apache.org/integration=<integration-name> -n camel-k --tail=50
```

### Follow logs in real-time
```bash
kubectl logs -l camel.apache.org/integration=<integration-name> -n camel-k -f
```

## Stopping/Deleting Integrations

### Delete a specific integration
```bash
kubectl delete integration <integration-name> -n camel-k
```

Example:
```bash
kubectl delete integration helloworld -n camel-k
```

This will automatically stop and remove the associated pod.

### Delete all integrations
```bash
kubectl delete integration --all -n camel-k
```

## Troubleshooting

### Check Camel K operator logs
```bash
kubectl logs deployment/camel-k-operator -n camel-k
```

### Check pod events
```bash
kubectl describe pod <pod-name> -n camel-k
```

### Check integration build status
```bash
kubectl get integrationkit -n camel-k
kubectl describe integrationkit <kit-name> -n camel-k
```

### Force pod restart
```bash
kubectl delete pod -l camel.apache.org/integration=<integration-name> -n camel-k
```

## Integration File Examples

### YAML DSL (helloworld.yaml)
```yaml
apiVersion: camel.apache.org/v1
kind: Integration
metadata:
  name: helloworld
spec:
  flows:
  - from:
      steps:
      - setBody:
          simple: Hello Camel from ${routeId}
      - log: ${body}
      uri: timer:yaml
```

### Java DSL - Inline (helloworld-inline.yaml)
```yaml
apiVersion: camel.apache.org/v1
kind: Integration
metadata:
  name: helloworld-inline
spec:
  sources:
    - name: HelloworldInline.java
      language: java
      content: |
        import org.apache.camel.builder.RouteBuilder;
        public class HelloworldInline extends RouteBuilder {
            @Override
            public void configure() {
                from("timer:inline")
                    .setBody().simple("Hello Camel Java from ${routeId} (inline)")
                    .log("${body}");
            }
        }
```

**Important:** The class name in the Java code must match the filename specified in the `name` field.

## Common Issues

### Integration stuck in "Building Kit"
- Check if the build process completed successfully
- Check operator logs for build errors
- Verify registry configuration in IntegrationPlatform

### Pod stuck in "ContainerCreating" with ConfigMap errors
- Use inline content approach instead of ConfigMap references
- Ensure the Java class name matches the filename in the Integration spec

### Java compilation errors
- Ensure the public class name matches the filename in the `name` field
- For class `HelloworldInline`, use `name: HelloworldInline.java`

## Cleanup

### Remove all integrations and resources
```bash
kubectl delete integration --all -n camel-k
kubectl delete integrationplatform --all -n camel-k
```

### Remove the namespace (if needed)
```bash
kubectl delete namespace camel-k
```
