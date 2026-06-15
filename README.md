Voici une version prête à coller dans un README.md, en gardant tout le contenu.

Stack Docker Compose locale : Grafana Mimir, MinIO, Nginx, Alloy et Grafana

Voici une version proche du TP officiel Grafana Mimir GitHub : 3 nœuds Mimir, MinIO, Nginx, Alloy, Grafana.

L’exemple officiel Grafana utilise aussi MinIO, 3 conteneurs Mimir, un load balancer Nginx sur 9009, et une config Mimir avec -config.file.

Pour éviter ton erreur précédente, je garde un seul bucket MinIO mimir, mais avec blocks_storage.storage_prefix: blocks, comme dans la config officielle.

⸻

Arborescence

mkdir -p config
touch docker-compose.yml config/mimir.yaml config/nginx.conf config/config.alloy

⸻

docker-compose.yml

services:
  minio:
    image: minio/minio:latest
    container_name: minio
    restart: unless-stopped
    entrypoint: [""]
    command: sh -c "mkdir -p /data/mimir && minio server /data --console-address ':9001'"
    environment:
      MINIO_ROOT_USER: admin
      MINIO_ROOT_PASSWORD: totototo
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - minio-data:/data
    networks:
      - observability
  mimir-1:
    image: grafana/mimir:latest
    container_name: mimir-1
    hostname: mimir-1
    restart: unless-stopped
    command:
      - "-config.file=/etc/mimir.yaml"
    volumes:
      - ./config/mimir.yaml:/etc/mimir.yaml:ro
      - mimir-1-data:/data
    networks:
      - observability
    depends_on:
      - minio
  mimir-2:
    image: grafana/mimir:latest
    container_name: mimir-2
    hostname: mimir-2
    restart: unless-stopped
    command:
      - "-config.file=/etc/mimir.yaml"
    volumes:
      - ./config/mimir.yaml:/etc/mimir.yaml:ro
      - mimir-2-data:/data
    networks:
      - observability
    depends_on:
      - minio
  mimir-3:
    image: grafana/mimir:latest
    container_name: mimir-3
    hostname: mimir-3
    restart: unless-stopped
    command:
      - "-config.file=/etc/mimir.yaml"
    volumes:
      - ./config/mimir.yaml:/etc/mimir.yaml:ro
      - mimir-3-data:/data
    networks:
      - observability
    depends_on:
      - minio
  nginx:
    image: nginx:latest
    container_name: nginx
    restart: unless-stopped
    ports:
      - "9009:9009"
    volumes:
      - ./config/nginx.conf:/etc/nginx/nginx.conf:ro
    networks:
      - observability
    depends_on:
      - mimir-1
      - mimir-2
      - mimir-3
  alloy:
    image: grafana/alloy:latest
    container_name: alloy
    restart: unless-stopped
    command:
      - run
      - /etc/alloy/config.alloy
    volumes:
      - ./config/config.alloy:/etc/alloy/config.alloy:ro
    networks:
      - observability
    depends_on:
      - nginx
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
    networks:
      - observability
    depends_on:
      - nginx
volumes:
  minio-data:
  mimir-1-data:
  mimir-2-data:
  mimir-3-data:
  grafana-data:
networks:
  observability:
    driver: bridge

⸻

config/mimir.yaml

# Config locale de test, inspirée de l'exemple officiel Grafana Mimir.
# Ne pas utiliser telle quelle en production.
target: all
multitenancy_enabled: false
server:
  http_listen_port: 8080
  grpc_listen_port: 9095
  log_level: warn
common:
  storage:
    backend: s3
    s3:
      endpoint: minio:9000
      access_key_id: admin
      secret_access_key: totototo
      insecure: true
      bucket_name: mimir
blocks_storage:
  storage_prefix: blocks
  tsdb:
    dir: /data/tsdb
compactor:
  data_dir: /data/compactor
ingester:
  ring:
    replication_factor: 3
    kvstore:
      store: memberlist
distributor:
  ring:
    kvstore:
      store: memberlist
store_gateway:
  sharding_ring:
    replication_factor: 3
    kvstore:
      store: memberlist
ruler:
  rule_path: /data/ruler
  alertmanager_url: http://127.0.0.1:8080/alertmanager
  ring:
    heartbeat_period: 2s
    heartbeat_timeout: 10s
    kvstore:
      store: memberlist
memberlist:
  join_members:
    - mimir-1
    - mimir-2
    - mimir-3
limits:
  ingestion_rate: 100000
  ingestion_burst_size: 200000

⸻

config/nginx.conf

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

⸻

config/config.alloy

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

La partie Alloy utilise prometheus.remote_write, composant officiel pour envoyer des métriques vers un endpoint compatible Prometheus remote write.

⸻

Lancement

cd "/chemin/vers/ton/projet"
docker compose up -d
docker ps

⸻

Logs utiles

docker logs minio --tail 50
docker logs mimir-1 --tail 100
docker logs mimir-2 --tail 100
docker logs mimir-3 --tail 100
docker logs nginx --tail 50
docker logs alloy --tail 50
docker logs grafana --tail 50

⸻

Accès

Grafana          : http://localhost:3000
MinIO API        : http://localhost:9000
MinIO console    : http://localhost:9001
Mimir via Nginx  : http://localhost:9009

⸻

Identifiants MinIO

admin / totototo

⸻

Debug Mimir

Si Mimir plante encore, vérifie d’abord :

docker logs mimir-1 --tail 100
docker compose config | grep -A5 -B5 mimir.yaml
ls -lh config/mimir.yaml
