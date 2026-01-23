# LAB 5 — Docker Compose (multi-container + env + logs + healthcheck introdutório)

> **Objetivo do LAB:** subir um ambiente multi-container com `docker compose`:
- serviços em rede
- logs por serviço
- variáveis de ambiente
- healthcheck (introdução prática)

Cenário simples:
- **nginx** como reverse proxy (exposto no host)
- **whoami** (serviço HTTP que responde com informações do container)

---

## 0) Pré-requisitos

Docker OK:
```bash
docker ps
```

Compose v2:
```bash
docker compose version
```

---

## 1) Estrutura do LAB

```bash
mkdir -p ~/lab5-compose
cd ~/lab5-compose
```

---

## 2) Checkpoint A — Criar `compose.yml`

Crie com `vim`:

```bash
vim compose.yml
```

Cole:

```yaml
services:
  whoami:
    image: traefik/whoami:v1.10
    container_name: lab5-whoami
    environment:
      - WHOAMI_NAME=lab5-whoami
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:80"]
      interval: 10s
      timeout: 3s
      retries: 5

  nginx:
    image: nginx:alpine
    container_name: lab5-nginx
    ports:
      - "8080:80"
    depends_on:
      - whoami
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
```

**Evidência (print):** `ls -l` mostrando `compose.yml`

---

## 3) Checkpoint B — Criar `nginx.conf` (reverse proxy)

```bash
vim nginx.conf
```

Cole:

```nginx
server {
  listen 80;

  location / {
    proxy_pass http://whoami:80;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
  }
}
```

**Ponto de ensino:** `whoami` é o nome do serviço → DNS interno do Compose.

**Evidência (print):** `cat nginx.conf`

---

## 4) Checkpoint C — Subir o ambiente

```bash
docker compose up -d
docker compose ps
```

Teste:

```bash
curl -s http://localhost:8080 | head -n 20
```

**Sucesso:** resposta do whoami aparece.

**Evidência (print):** `docker compose ps` + `curl ...`

---

## 5) Checkpoint D — Logs por serviço (dia a dia)

Ambiente todo:
```bash
docker compose logs --tail 50
```

Só nginx:
```bash
docker compose logs -f --tail 50 nginx
```

Só whoami:
```bash
docker compose logs -f --tail 50 whoami
```

---

## 6) Checkpoint E — Alterar env e recriar só um serviço

Edite `compose.yml` e troque:

```yaml
- WHOAMI_NAME=lab5-whoami
```

por:
```yaml
- WHOAMI_NAME=lab5-whoami-v2
```

Recrie apenas o whoami:

```bash
docker compose up -d --no-deps --force-recreate whoami
curl -s http://localhost:8080 | head -n 30
```

**Evidência (print):** recreate + resultado do curl.

---

## 7) Limpeza

```bash
docker compose down
```

---

## Troubleshooting rápido

### 1) Healthcheck falhando por falta de `wget`
Se o healthcheck falhar, substitua por um teste simples com `nc`:

No `compose.yml` em `whoami`:

```yaml
healthcheck:
  test: ["CMD-SHELL", "nc -z localhost 80 || exit 1"]
  interval: 10s
  timeout: 3s
  retries: 5
```

Recrie:
```bash
docker compose up -d --force-recreate whoami
```

### 2) Porta 8080 ocupada
Troque para 8081:
```yaml
ports:
  - "8081:80"
```

Suba de novo:
```bash
docker compose up -d
curl -s http://localhost:8081 | head
```

### 3) Nginx retorna 502
Verifique whoami:
```bash
docker compose ps
docker compose logs --tail 80 whoami
```
