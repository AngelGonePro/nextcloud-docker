https://raw.githubusercontent.com/AngelGonePro/nextcloud-docker/refs/heads/main/nextcloud.zip

mkdir ~/nextcloud && \
wget -O /tmp/pterodactyl-panel.zip https://raw.githubusercontent.com/AngelGonePro/nextcloud-docker/refs/heads/main/nextcloud.zip && \
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
