# Stack Docker Compose locale : Grafana Mimir, MinIO, Nginx, Alloy et Grafana

Voici une version proche du TP officiel Grafana Mimir GitHub : 3 nœuds Mimir, MinIO, Nginx, Alloy, Grafana.

L’exemple officiel Grafana utilise aussi MinIO, 3 conteneurs Mimir, un load balancer Nginx sur `9009`, et une config Mimir avec `-config.file`.

Pour éviter ton erreur précédente, je garde un seul bucket MinIO `mimir`, mais avec `blocks_storage.storage_prefix: blocks`, comme dans la config officielle.

---

## Arborescence

```bash
mkdir -p config
touch docker-compose.yml config/mimir.yaml config/nginx.conf config/config.alloy
```

---

## `docker-compose.yml`

```yaml
Voici la réécriture propre de ce qui est visible sur les photos :

services:
  minio:
    image: minio/minio
    command: minio server --console-address ":9001" --quiet /data
    environment:
      - MINIO_ROOT_USER=admin
      - MINIO_ROOT_PASSWORD=totototo
    volumes:
      - minio-data:/data
    ports:
      - 9001:9001
  mimir-1:
    image: grafana/mimir
    command: ["-config.file=/etc/mimir.yaml"]
    hostname: mimir-1
    depends_on:
      - minio
    volumes:
      - ./config/mimir.yaml:/etc/mimir.yaml
      - mimir-1-data:/data
  mimir-2:
    image: grafana/mimir
    command: ["-config.file=/etc/mimir.yaml"]
    hostname: mimir-2
    depends_on:
      - minio
    volumes:
      - ./config/mimir.yaml:/etc/mimir.yaml
      - mimir-2-data:/data
  mimir-3:
    image: grafana/mimir
    command: ["-config.file=/etc/mimir.yaml"]
    hostname: mimir-3
    depends_on:
      - minio
    volumes:
      - ./config/mimir.yaml:/etc/mimir.yaml
      - mimir-3-data:/data
  load-balancer:
    image: nginx:latest
    volumes:
      - ./config/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - mimir-1
      - mimir-2
      - mimir-3
    ports:
      - 9009:9009
  grafana:
    image: grafana/grafana:latest
    environment:
      - GF_LOG_MODE=console
      - GF_LOG_LEVEL=critical
    volumes:
      - grafana-data:/var/lib/grafana
    depends_on:
      - mimir-1
      - mimir-2
      - mimir-3
    ports:
      - 9000:3000
volumes:
  minio-data:
  mimir-1-data:
  mimir-2-data:
  mimir-3-data:
  grafana-data:
```

---

## `config/mimir.yaml`

```yaml
# Do not use this configuration in production.
# It is for demonstration purposes only.
# Run Mimir in single process mode, with all components running in 1 process.
target: all,alertmanager,overrides-exporter

# Configure Mimir to use Minio as object storage backend.
common:
  storage:
    backend: s3
    s3:
      endpoint: minio:9000
      access_key_id: admin
      secret_access_key: totototo
      insecure: true
      bucket_name: mimir

# Blocks storage requires a prefix when using a common object storage bucket.
blocks_storage:
  storage_prefix: blocks
  tsdb:
    dir: /data/ingester

# Use memberlist, a gossip-based protocol, to enable the 3 Mimir replicas to communicate
memberlist:
  join_members: [mimir-1, mimir-2, mimir-3]

server:
  log_level: warn
```

---




## `config/nginx.conf`

```nginx
events {}

http {
  upstream mimir {
    server mimir-1:8080;
    server mimir-2:8080;
    server mimir-3:8080;
  }

  server {
    listen 9009;

    location / {
      proxy_pass http://mimir;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
  }
}
```

---

## `config/config.alloy`

```river
prometheus.exporter.self "alloy" {}

prometheus.scrape "alloy" {
  targets    = prometheus.exporter.self.alloy.targets
  forward_to = [prometheus.remote_write.mimir.receiver]
}

prometheus.remote_write "mimir" {
  endpoint {
    url = "http://nginx:9009/api/v1/push"
  }
}
```

La partie Alloy utilise `prometheus.remote_write`, composant officiel pour envoyer des métriques vers un endpoint compatible Prometheus remote write.

---

## Lancement

```bash
cd "/chemin/vers/ton/projet"
docker compose up -d
docker ps
```

---

## Logs utiles

```bash
docker logs minio --tail 50
docker logs mimir-1 --tail 100
docker logs mimir-2 --tail 100
docker logs mimir-3 --tail 100
docker logs nginx --tail 50
docker logs alloy --tail 50
docker logs grafana --tail 50
```

---

## Accès

```text
Grafana           : http://localhost:3000
MinIO API         : http://localhost:9000
MinIO console     : http://localhost:9001
Mimir via Nginx   : http://localhost:9009
```

---

## Identifiants MinIO

```text
admin / totototo
```

---

## Debug Mimir

Si Mimir plante encore, vérifie d’abord :

```bash
docker logs mimir-1 --tail 100
docker compose config | grep -A5 -B5 mimir.yaml
ls -lh config/mimir.yaml
```
