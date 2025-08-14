---
alias: update script n8n
---

Этот скрипт:
1. Автоматически определяет путь к данным DATA_PATH
2. Создает папку для бэкапов данных `.n8n`
3. Обновляет образ n8n
4. Останавливает и удаляет старый контейнер n8n
5. Запускает новый контейнер n8n с обновлённым образом
6. Проверяет новую версию

## Script: **`update-n8n.sh`**  с автоопределением DATA_PATH
```bash
#!/bin/bash
set -e

# === НАСТРОЙКИ ===
CONTAINER_NAME="n8n"
IMAGE_NAME="n8nio/n8n:latest"
HOST="your-domain.com"
WEBHOOK_URL="https://your-domain.com/"
PORT=5678
BACKUP_DIR="$HOME/n8n-backups"
MAX_BACKUPS=5  # сколько последних бэкапов хранить

# === АВТООПРЕДЕЛЕНИЕ ПУТИ К ДАННЫМ ===
echo "[0/6] Определяем путь к данным..."
DATA_PATH=$(sudo docker inspect -f '{{ range .Mounts }}{{ if eq .Destination "/root/.n8n" }}{{ .Source }}{{ end }}{{ end }}' "$CONTAINER_NAME" 2>/dev/null || true)

if [ -z "$DATA_PATH" ]; then
  echo "❌ Не удалось автоматически определить DATA_PATH."
  echo "   Убедитесь, что контейнер $CONTAINER_NAME существует и смонтирован /root/.n8n"
  exit 1
fi

echo "   Найден путь к данным: $DATA_PATH"

# === СОЗДАНИЕ ПАПКИ ДЛЯ БЭКАПОВ ===
mkdir -p "$BACKUP_DIR"

# === БЭКАП ===
BACKUP_FILE="$BACKUP_DIR/n8n-backup-$(date +%F-%H%M).tar.gz"
echo "[1/6] Создаём бэкап данных в $BACKUP_FILE..."
tar -czf "$BACKUP_FILE" -C "$DATA_PATH" .

# === ОЧИСТКА СТАРЫХ БЭКАПОВ ===
echo "[2/6] Очищаем старые бэкапы, оставляем только $MAX_BACKUPS..."
ls -t "$BACKUP_DIR"/n8n-backup-*.tar.gz | tail -n +$((MAX_BACKUPS+1)) | xargs -r rm --

# === ОБНОВЛЕНИЕ ОБРАЗА ===
echo "[3/6] Загружаем последнюю версию образа n8n..."
sudo docker pull $IMAGE_NAME

# === ОСТАНОВКА И УДАЛЕНИЕ СТАРОГО КОНТЕЙНЕРА ===
if sudo docker ps -a --format '{{.Names}}' | grep -q "^$CONTAINER_NAME$"; then
  echo "[4/6] Останавливаем и удаляем старый контейнер..."
  sudo docker stop $CONTAINER_NAME || true
  sudo docker rm $CONTAINER_NAME || true
else
  echo "[4/6] Контейнер не найден — пропускаем остановку."
fi

# === ЗАПУСК ОБНОВЛЁННОГО КОНТЕЙНЕРА ===
echo "[5/6] Запускаем n8n с обновлённым образом..."
sudo docker run -d --restart unless-stopped -it \
  --name $CONTAINER_NAME \
  -p $PORT:5678 \
  -e N8N_HOST="$HOST" \
  -e WEBHOOK_TUNNEL_URL="$WEBHOOK_URL" \
  -e WEBHOOK_URL="$WEBHOOK_URL" \
  -v $DATA_PATH:/root/.n8n \
  $IMAGE_NAME

# === ПРОВЕРКА ВЕРСИИ ===
echo "[6/6] Проверка новой версии..."
sudo docker exec -it $CONTAINER_NAME n8n --version

echo "✅ Обновление завершено!"
```
## How to use it

1. Save as:
```bash
nano update-n8n.sh
```
Paste the script, change:
- `your-domain.com` to your actual domain    
- port if different    
- `$HOME/.n8n` if your [[data path]] is custom

2. Make executable:
```bash
chmod +x update-n8n.sh
```
3. Run:
```bash
./update-n8n.sh
```
This will **always back up your data first** and allow quick rollback if something breaks.

## Добавление в скрипт логики очистки старых бэкапов

Например, будем хранить только последние **5 архивов**.

Обновлённый скрипт `update-n8n.sh`
```bash
#!/bin/bash
set -e

# === НАСТРОЙКИ ===
CONTAINER_NAME="n8n"
IMAGE_NAME="n8nio/n8n:latest"
DATA_PATH="$HOME/.n8n"   # путь к данным n8n
HOST="your-domain.com"
WEBHOOK_URL="https://your-domain.com/"
PORT=5678
BACKUP_DIR="$HOME/n8n-backups"
MAX_BACKUPS=5  # сколько последних бэкапов хранить

# === СОЗДАНИЕ ПАПКИ ДЛЯ БЭКАПОВ ===
mkdir -p "$BACKUP_DIR"

# === БЭКАП ===
BACKUP_FILE="$BACKUP_DIR/n8n-backup-$(date +%F-%H%M).tar.gz"
echo "[1/6] Создаём бэкап данных в $BACKUP_FILE..."
tar -czf "$BACKUP_FILE" -C "$DATA_PATH" .

# === ОЧИСТКА СТАРЫХ БЭКАПОВ ===
echo "[2/6] Очищаем старые бэкапы, оставляем только $MAX_BACKUPS..."
ls -t "$BACKUP_DIR"/n8n-backup-*.tar.gz | tail -n +$((MAX_BACKUPS+1)) | xargs -r rm --

# === ОБНОВЛЕНИЕ ИЗОБРАЖЕНИЯ ===
echo "[3/6] Загружаем последнюю версию образа n8n..."
sudo docker pull $IMAGE_NAME

# === ОСТАНОВКА И УДАЛЕНИЕ СТАРОГО КОНТЕЙНЕРА ===
if sudo docker ps -a --format '{{.Names}}' | grep -q "^$CONTAINER_NAME$"; then
  echo "[4/6] Останавливаем и удаляем старый контейнер..."
  sudo docker stop $CONTAINER_NAME || true
  sudo docker rm $CONTAINER_NAME || true
else
  echo "[4/6] Контейнер не найден — пропускаем остановку."
fi

# === ЗАПУСК ОБНОВЛЁННОГО КОНТЕЙНЕРА ===
echo "[5/6] Запускаем n8n с обновлённым образом..."
sudo docker run -d --restart unless-stopped -it \
  --name $CONTAINER_NAME \
  -p $PORT:5678 \
  -e N8N_HOST="$HOST" \
  -e WEBHOOK_TUNNEL_URL="$WEBHOOK_URL" \
  -e WEBHOOK_URL="$WEBHOOK_URL" \
  -v $DATA_PATH:/root/.n8n \
  $IMAGE_NAME

# === ПРОВЕРКА ВЕРСИИ ===
echo "[6/6] Проверка новой версии..."
sudo docker exec -it $CONTAINER_NAME n8n --version

echo "✅ Обновление завершено!"
```
### Что изменилось

- Все бэкапы хранятся в папке `~/n8n-backups`    
- При каждом обновлении старые бэкапы удаляются, оставляя только **5 последних**    
- Логика шагов осталась прежней, но нумерация изменилась с 5 на 6 шагов