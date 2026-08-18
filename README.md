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
```
cat > ~/nextcloud/nginx/nextcloud.conf << 'EOF'
map $http_x_forwarded_proto $nc_forwarded_proto {
    default $http_x_forwarded_proto;
    ''      $scheme;
}

map $nc_forwarded_proto $nc_https {
    https on;
    default off;
}

server {
    listen 80;
    server_name _;

    root /var/www/html;
    index index.php index.html;

    client_max_body_size 10G;
    client_body_timeout 3600s;
    fastcgi_buffers 64 4K;

    add_header Strict-Transport-Security "max-age=15768000; includeSubDomains" always;
    add_header X-Content-Type-Options nosniff always;
    add_header X-Frame-Options SAMEORIGIN always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Robots-Tag none always;
    add_header X-Download-Options noopen always;
    add_header X-Permitted-Cross-Domain-Policies none always;
    add_header Referrer-Policy no-referrer always;

    location = /robots.txt { access_log off; log_not_found off; }
    location = /.well-known/carddav { return 301 $scheme://$host/remote.php/dav; }
    location = /.well-known/caldav  { return 301 $scheme://$host/remote.php/dav; }
    location = /.well-known/webfinger { return 301 $scheme://$host/index.php/.well-known/webfinger; }
    location = /.well-known/nodeinfo  { return 301 $scheme://$host/index.php/.well-known/nodeinfo; }

    location ~ ^/(?:build|tests|config|lib|3rdparty|templates|data)/ { deny all; }
    location ~ ^/(?:\.|autotest|occ|issue|indie|db_|console) { deny all; }

    location ~ ^/(status|ping)$ {
        access_log off;
        allow 127.0.0.1;
        deny all;
        include fastcgi_params;
        fastcgi_pass app:9000;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }

    location / {
        try_files $uri $uri/ /index.php$request_uri;
    }

    location ~ \.php(?:$|/) {
        fastcgi_split_path_info ^(.+\.php)(/.*)$;
        fastcgi_pass app:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
        fastcgi_param HTTPS $nc_https;
        fastcgi_param HTTP_X_FORWARDED_FOR $proxy_add_x_forwarded_for;
        fastcgi_param HTTP_X_FORWARDED_PROTO $nc_forwarded_proto;
        fastcgi_read_timeout 3600;
    }

    location ~ \.mjs$ {
        types { text/javascript mjs; }
        default_type text/javascript;
        try_files $uri /index.php$request_uri;
        expires 6M;
        access_log off;
    }

    location ~ \.(?:css|js|svg|gif|png|jpg|ico|woff2?)$ {
        try_files $uri /index.php$request_uri;
        expires 6M;
        access_log off;
    }
}
EOF

grep -c "mjs\|webfinger" ~/nextcloud/nginx/nextcloud.conf
docker compose exec web nginx -t
docker compose exec web nginx -s reload
docker compose exec web cat /etc/nginx/conf.d/default.conf | grep -c "mjs\|webfinger"
curl -I http://10.0.0.50:8080/index.php/apps/theming/theme/light.mjs 2>&1 | grep -i content-type
curl -o /dev/null -s -w "%{http_code}\n" http://10.0.0.50:8080/.well-known/webfinger
```
```
grep -c "nc_https" ~/nextcloud/nginx/nextcloud.conf
docker compose exec web nginx -t
docker compose exec web nginx -s reload
docker compose exec web cat /etc/nginx/conf.d/default.conf | grep -c "nc_https"
```
```
docker compose exec -u www-data app php occ db:add-missing-indices
docker compose exec -u www-data app php occ config:system:set maintenance_window_start --type=integer --value=1
docker compose exec -u www-data app php occ config:system:set default_phone_region --value="US"
```
403 means not exposed: `curl -o /dev/null -s -w "%{http_code}\n" http://10.0.0.50:8080/data/.ocdata`
