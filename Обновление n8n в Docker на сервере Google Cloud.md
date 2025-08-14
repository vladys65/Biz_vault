1️⃣ Check your current n8n version
```bash
sudo docker exec -it n8n n8n --version
```
This way you’ll know what version you’re on before updating.


2️⃣ Backup your n8n data
```bash
tar -czvf n8n-backup-$(date +%F).tar.gz ~/.n8n
```

3️⃣ Pull the latest n8n Docker image
```bash
sudo docker pull n8nio/n8n:latest
```
*(You can specify a version instead of `latest`, e.g. `n8nio/n8n:1.68.0` if you want to lock it.)*


4️⃣ Stop and remove the old container
```bash
sudo docker stop n8n
sudo docker rm n8n
```
This does **not** delete your workflows because they’re stored in the mounted volume (`-v ~/.n8n:/root/.n8n`).


5️⃣ Start the new container
```bash
sudo docker run -d --restart unless-stopped -it \
  --name n8n \
  -p 5678:5678 \
  -e N8N_HOST="your-domain.com" \
  -e WEBHOOK_TUNNEL_URL="https://your-domain.com/" \
  -e WEBHOOK_URL="https://your-domain.com/" \
  -v ~/.n8n:/root/.n8n \
  n8nio/n8n:latest
```

6️⃣ Verify the update
```bash
sudo docker exec -it n8n n8n --version
```
It should now show the new version.

💡 **Tip**  You can make an [[Скрипт обновления n8n в Docker на сервере Google Cloud|update script n8n]] so it’s just one command to pull, restart, and check the version.
