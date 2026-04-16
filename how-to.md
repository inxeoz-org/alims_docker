

# Frappe Docker Setup (Custom Image + App)

Based on frappe_docker

---

## 1. Clone Repository

```bash

git clone --depth 1 https://github.com/inxeoz-org/alims_docker
cd alims_docker
```

---

## 2. Define Apps (`apps.json`)

```json
[
  {
    "url": "https://github.com/frappe/erpnext",
    "branch": "version-15"
  }
]
```

---

## 3. Build Custom Image

### Recommended (BuildKit)

```bash
docker build \
  --build-arg=FRAPPE_PATH=https://github.com/frappe/frappe \
  --build-arg=FRAPPE_BRANCH=version-15 \
  --secret=id=apps_json,src=apps.json \
  --tag=custom:15 \
  --file=images/layered/Containerfile .
```

### For Older Docker Versions

```bash
export APPS_JSON_BASE64=$(base64 -w 0 apps.json)

docker build \
  --build-arg=FRAPPE_PATH=https://github.com/frappe/frappe \
  --build-arg=FRAPPE_BRANCH=version-15 \
  --build-arg=APPS_JSON_BASE64=$APPS_JSON_BASE64 \
  --tag=custom:15 \
  --file=images/layered/Containerfile .
```

---

## 4. Environment Configuration (`custom.env`)

```env
ERPNEXT_VERSION=v15
DB_PASSWORD=123
FRAPPE_SITE_NAME_HEADER=alis.localhost
HTTP_PUBLISH_PORT=8080

CUSTOM_IMAGE=custom
CUSTOM_TAG=15
PULL_POLICY=missing
```

---

## 5. Generate Compose File

```bash
docker compose --env-file custom.env \
  -f compose.yaml \
  -f overrides/compose.mariadb.yaml \
  -f overrides/compose.redis.yaml \
  -f overrides/compose.noproxy.yaml \
  config > compose.custom.yaml
```

---

## 6. Reset Existing Deployment

```bash
docker compose -p alis -f compose.custom.yaml down -v
```

---

## 7. Start Containers

```bash
docker compose -p alis -f compose.custom.yaml up -d
```

---

## 8. Verify Containers

```bash
docker ps | grep alis
```

Expected: containers running with port mapping `8080 → 8080`

---

## 9. Setup Site and Install App

### Enter Backend Container

```bash
docker compose -p alis -f compose.custom.yaml exec backend bash
```

---

### Create Site

```bash
bench new-site alis.localhost
```

When prompted:

```
MariaDB root user: root
Password: 123
```

db passsword from ``custom.env``

---

### Set Default Site

```bash
bench use alis.localhost
```

---

### Install App

```bash
bench install-app erpnext
```

---

### Restart

```bash
bench restart
```

---

## 10. Access Application

Test locally:

```bash
curl localhost:8080 | head -10
```

If response shows `"does not exist"`:

```bash
curl -H "Host: alis.localhost" localhost:8080 | head -10
```

---

## 11. Network Access

```bash
curl <public_or_network_ip>:8080 | head -10
```

or

```bash
curl -H "Host: alis.localhost" <public_or_network_ip>:8080 | head -10
```

Examples:

```
172.18.x.x
10.x.x.x
```

---

## 12. Open Firewall (RHEL/CentOS)

```bash
sudo firewall-cmd --zone=public --add-port=8080/tcp --permanent
sudo firewall-cmd --reload
```

---

## Troubleshooting

### Site Not Resolving

Use Host header:

```bash
curl -H "Host: alis.localhost" localhost:8080
```

---

### Database Connection Issues

Run inside backend container:

```bash
bench new-site wiki.localhost \
  --db-host mariadb \
  --mariadb-root-password 123 \
  --admin-password admin_password_here
```

---

### If Still Not Working

* Verify containers are running
* Check logs:

  ```bash
  docker compose -p alis -f compose.custom.yaml logs
  ```
* Ensure port `8080` is open
