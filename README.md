# Apex Gateway

The Apex Gateway is a centralized Nginx reverse proxy designed to manage traffic for multiple applications on a single virtual machine. It follows the **Apex App Standard (AAS-1.0)**, allowing services to be routed via sub-paths (e.g., `hostname:10000/feedback/`).

## How it Works

1.  **Single Entry Point**: The gateway listens on port `10000` and routes requests to internal Docker containers.
2.  **Service Discovery**: It uses a shared Docker network (`apex-internal`) to communicate with other containers by their names.
3.  **Modular Configuration**: Routing rules for each application are stored in the `services/` directory and automatically loaded by Nginx.

## Adding New Services

To add a new service to the system, follow these two steps:

### 1. Update the Gateway Configuration
Add a new `location` block to the system. These are managed in `services/gateway.conf`.

Example block for a new service called `parking-app`:
```nginx
location /parking/ {
    # Use the container name defined in the app's docker-compose.yml
    set $upstream http://parking-app:8080;
    proxy_pass $upstream/; 
    proxy_set_header X-Forwarded-Prefix /parking;

    # Standard proxy headers (AAS-1.0)
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

> [!NOTE]
> We use the `set $upstream` variable pattern. This prevents Nginx from crashing if the downstream container is temporarily down during startup. This requires the `resolver 127.0.0.11` line present in the main server block.

### 2. Configure the Service's `docker-compose.yml`
For the gateway to route traffic to your service, the service must be on the same Docker network.

```yaml
services:
  your-service-name:
    container_name: parking-app  # This name MUST match the one in Nginx config
    # ... your other config ...
    networks:
      - apex-internal

networks:
  apex-internal:
    external: true
    name: apex-internal
```

## Requirements for Services (AAS-1.0)

To work seamlessly with the gateway, applications should:
*   **Port**: Listen on internal port `8080` (the standard for Apex applications).
*   **Networking**: Use the `apex-internal` bridge network.
*   **Path Handling**: Respect the `BASE_PATH` environment variable or the `X-Forwarded-Prefix` header so that links and assets load correctly from the sub-path (e.g., `/parking/static/style.css` instead of `/static/style.css`).

## Deployment
To apply changes to the gateway after adding a configuration:
```powershell
docker exec apex-gateway nginx -s reload
```
