# posthog-selfhost-cheatsheet

Русская шпаргалка по разворачиванию self-hosted PostHog на Ubuntu 24.04 LTS через официальный Hobby Docker Compose installer.

> Цель: быстро поднять PostHog на своей VM, не потерять доступ по SSH, не забить диск ClickHouse/Docker-логами и понимать, где смотреть проблемы.

## Проверенная конфигурация

- Ubuntu 24.04 LTS
- CPU: 4 cores
- RAM: 16 GB
- Disk: изначально 30 GB SSD
- Deployment: official PostHog Hobby installer
- Access: по IP или через локальную запись в `hosts`

Практический вывод по диску:

- `30 GB` формально может быть указан как минимум, но на практике этого мало.
- Для реальной установки лучше считать минимумом `80 GB`, комфортнее `100 GB+`.
- ClickHouse, Docker images, containerd snapshots и ingestion-данные быстро съедают место.

## 1. Проверка диска и LVM

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
df -h /
pvs
vgs
lvs
```

Если `VFree` в `vgs` равен `0`, значит свободное LVM-место уже отдано root-разделу.

После увеличения диска у провайдера:

```bash
apt install -y cloud-guest-utils

growpart /dev/sda 3
pvresize /dev/sda3
lvextend -r -l +100%FREE /dev/vg0/lv-0

df -h /
vgs
lvs
```

Для NVMe:

```bash
growpart /dev/nvme0n1 3
pvresize /dev/nvme0n1p3
lvextend -r -l +100%FREE /dev/vg0/lv-0
```

## 2. Firewall через UFW

```bash
ufw default deny incoming
ufw default allow outgoing

ufw allow 22/tcp comment 'SSH'
ufw allow 80/tcp comment 'PostHog HTTP'
ufw allow 443/tcp comment 'HTTPS'

ufw --force enable
ufw status verbose
```

Если `ufw enable` падает с ошибкой кодировки Python, используй:

```bash
ufw --force enable
```

Проверить слушающие порты:

```bash
ss -tulpen
```

Важно: Docker может публиковать порты через свои iptables-правила. Проверяй не только UFW, но и:

```bash
docker compose ps
ss -tulpen
```

## 3. Docker Engine без Snap

```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do
  apt remove -y "$pkg" || true
done

snap remove docker || true

apt update
apt install -y ca-certificates curl gnupg

install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc

chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" \
  > /etc/apt/sources.list.d/docker.list

apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

systemctl enable --now docker

docker version
docker compose version
```

## 4. Ограничение Docker-логов

Обязательно для маленького диска:

```bash
cat > /etc/docker/daemon.json <<'EOF'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "3"
  },
  "live-restore": true
}
EOF

systemctl restart docker
docker info | grep -i "Logging Driver"
```

Ожидаемо:

```text
Logging Driver: json-file
```

## 5. Установка PostHog Hobby

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/posthog/posthog/HEAD/bin/deploy-hobby)"
```

Если спрашивает версию PostHog, можно нажать `Enter`, будет использован `latest`.

Если спрашивает Anthropic/OpenAI API key, можно нажать `Enter`, AI-функции можно добавить позже.

### Важное про IP

Official installer ожидает домен, а не IP и не URL.

Если всё же ставишь для доступа по IP, не вводи:

```text
http://195.50.4.248
```

Вводи только:

```text
195.50.4.248
```

Иначе можно получить кривую строку:

```text
https://http://195.50.4.248
```

## 6. Где лежит конфиг PostHog

Если установка выполнялась из `/opt/posthog`:

```bash
cd /opt/posthog
ls -la
```

Ожидаемые файлы:

```text
.env
docker-compose.yml
docker-compose.base.yml
.env.services
compose/
posthog/
share/
```

Проверить URL-настройки:

```bash
grep -a -n "DOMAIN\|CADDY_HOST\|TLS_BLOCK\|SITE_URL\|REGISTRY_URL\|POSTHOG_APP_TAG" .env
```

Для доступа по HTTP через IP:

```bash
DOMAIN=195.50.4.248
TLS_BLOCK=
REGISTRY_URL=posthog/posthog
CADDY_TLS_BLOCK=
CADDY_HOST=http://195.50.4.248
POSTHOG_APP_TAG=latest
SITE_URL=http://195.50.4.248
```

Секреты не менять без причины:

```bash
POSTHOG_SECRET=...
ENCRYPTION_SALT_KEYS=...
BROWSERLESS_SECRET=...
```

Если изменить `POSTHOG_SECRET` или `ENCRYPTION_SALT_KEYS` на живой установке, можно сломать сессии и зашифрованные данные.

После правки `.env`:

```bash
docker compose down
docker compose up -d
```

Проверка:

```bash
docker compose ps
curl -I http://195.50.4.248
```

## 7. Локальный hosts вместо домена

На Windows можно добавить локальное имя, например `posthog.test`.

Файл:

```text
C:\Windows\System32\drivers\etc\hosts
```

Строка:

```text
195.50.4.248 posthog.test
```

После сохранения:

```powershell
ipconfig /flushdns
ping posthog.test
```

Тогда на сервере в `/opt/posthog/.env` можно использовать:

```bash
DOMAIN=posthog.test
TLS_BLOCK=
CADDY_TLS_BLOCK=
CADDY_HOST=http://posthog.test
SITE_URL=http://posthog.test
```

Открывать:

```text
http://posthog.test
```

Это работает только на компьютерах, где прописан `hosts`.

## 8. Диагностика 502 Bad Gateway

Если Caddy отвечает:

```text
HTTP/1.1 502 Bad Gateway
Server: Caddy
```

Значит proxy работает, но backend `web` ещё не готов или Caddy смотрит не туда.

Команды:

```bash
cd /opt/posthog

curl -I http://localhost/_health
docker compose ps
docker compose logs --tail=100 web
docker compose logs --tail=100 proxy
docker compose logs --tail=100 worker
```

После рестарта PostHog может подниматься несколько минут из-за миграций.

## 9. Если установка упала с no space left on device

Типичная ошибка:

```text
failed to extract layer ... no space left on device
Failed to start stack after 3 attempts
```

Проверить место:

```bash
df -h /
docker system df
du -sh /var/lib/docker /var/lib/containerd 2>/dev/null || true
```

Если установка не завершилась, контейнеров и volumes нет, можно чистить:

```bash
docker system prune -af
systemctl restart docker containerd

df -h /
docker system df
du -sh /var/lib/docker /var/lib/containerd 2>/dev/null || true
```

Если контейнеров и volumes точно нет, но containerd всё ещё занимает много места:

```bash
systemctl stop docker containerd

rm -rf /var/lib/containerd/*
rm -rf /var/lib/docker/*

systemctl start containerd docker
```

Не запускай `docker volume prune` на живой установке PostHog: можно удалить данные ClickHouse/Postgres.

## 10. Ревизия ресурсов

Общее состояние сервера:

```bash
hostnamectl
uptime
free -h
df -h /
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
```

Docker:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
docker stats --no-stream
docker system df
du -sh /var/lib/docker /var/lib/containerd /opt/posthog 2>/dev/null || true
```

PostHog:

```bash
cd /opt/posthog
docker compose ps
docker compose top
```

Кто ест RAM/CPU:

```bash
ps aux --sort=-%mem | head -20
ps aux --sort=-%cpu | head -20
```

ClickHouse:

```bash
CLICKHOUSE_CONTAINER="posthog-clickhouse-1"

docker exec "$CLICKHOUSE_CONTAINER" clickhouse-client --query "
SELECT
    formatReadableSize(sum(bytes_on_disk)) AS total_on_disk
FROM system.parts
WHERE active;
"

docker exec "$CLICKHOUSE_CONTAINER" clickhouse-client --query "
SELECT
    database,
    table,
    formatReadableSize(sum(bytes_on_disk)) AS size,
    sum(rows) AS rows
FROM system.parts
WHERE active
GROUP BY database, table
ORDER BY sum(bytes_on_disk) DESC
LIMIT 20;
"
```

Логи:

```bash
journalctl --disk-usage
find /var/lib/docker/containers -name "*-json.log" -type f -printf "%s %p\n" 2>/dev/null | sort -nr | head -20
```

## 11. Ограничение systemd journal

```bash
mkdir -p /etc/systemd/journald.conf.d

cat > /etc/systemd/journald.conf.d/99-size-limit.conf <<'EOF'
[Journal]
SystemMaxUse=500M
MaxRetentionSec=7day
EOF

systemctl restart systemd-journald
journalctl --disk-usage
```

## 12. Безопасная чистка Docker-мусора

Без удаления volumes:

```bash
cat > /usr/local/sbin/docker-safe-cleanup.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

docker image prune -af --filter "until=168h"
docker builder prune -af --filter "until=168h"
journalctl --vacuum-time=7d
EOF

chmod +x /usr/local/sbin/docker-safe-cleanup.sh
```

Ручной запуск:

```bash
/usr/local/sbin/docker-safe-cleanup.sh
```

## 13. Быстрый checklist после установки

```bash
ufw status verbose
ss -tulpen
docker compose ps
docker stats --no-stream
df -h /
docker system df
journalctl --disk-usage
curl -I http://localhost/_health
```

## Главные выводы

- Для PostHog на Docker лучше сразу выделять `80-100 GB+` диска.
- Не менять `POSTHOG_SECRET` и `ENCRYPTION_SALT_KEYS` после запуска.
- Не использовать `docker volume prune` на живой установке.
- Для IP-доступа проще использовать HTTP или локальный `hosts`.
- Для нормального HTTPS нужен настоящий домен и DNS A-record на VM.
- ClickHouse и Docker/containerd нужно регулярно мониторить по диску.
````
