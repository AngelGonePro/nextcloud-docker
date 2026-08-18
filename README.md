https://raw.githubusercontent.com/AngelGonePro/nextcloud-docker/refs/heads/main/nextcloud.zip
```
mkdir ~/nextcloud && \
wget -O /tmp/nextcloud.zip https://raw.githubusercontent.com/AngelGonePro/nextcloud-docker/refs/heads/main/nextcloud.zip && \
python3 - << 'EOF'
import zipfile, os

zip_path = "/tmp/nextcloud.zip"
extract_to = "nextcloud"

with zipfile.ZipFile(zip_path) as z:
    for member in z.namelist():
        parts = member.split("/", 1)
        if len(parts) > 1:
            target = os.path.join(extract_to, parts[1])
            if not member.endswith("/"):
                os.makedirs(os.path.dirname(target), exist_ok=True)
                with open(target, "wb") as f:
                    f.write(z.read(member))
EOF
rm /tmp/nextcloud.zip
```
```
cd ~/nextcloud
cp .env.example .env
nano .env      # set NC_TRUSTED_PROXIES to your proxy VM's IP, set NC_PORT
```
```
mkdir data
```
```
docker compose up -d --force-recreate app cron

until docker compose exec -T -u www-data app php occ status 2>/dev/null | grep -q "installed: true"; do sleep 2; done

docker compose exec -u www-data app php occ app:enable files_external
docker compose exec -u www-data app php occ files_external:create "Shared Media" local null::null -c datadir=/mnt/media
docker compose exec -u www-data app php occ files_external:option 1 filesystem_check_changes 1
```
```
cd ~/nextcloud
docker compose up -d web
docker compose exec -u www-data app php occ config:system:delete overwriteprotocol
```
