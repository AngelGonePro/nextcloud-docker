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
For powershell one liner:
```
echo "bWFwICRodHRwX3hfZm9yd2FyZGVkX3Byb3RvICRuY19mb3J3YXJkZWRfcHJvdG8gewogICAgZGVmYXVsdCAkaHR0cF94X2ZvcndhcmRlZF9wcm90bzsKICAgICcnICAgICAgJHNjaGVtZTsKfQoKbWFwICRuY19mb3J3YXJkZWRfcHJvdG8gJG5jX2h0dHBzIHsKICAgIGh0dHBzIG9uOwogICAgZGVmYXVsdCBvZmY7Cn0KCnNlcnZlciB7CiAgICBsaXN0ZW4gODA7CiAgICBzZXJ2ZXJfbmFtZSBfOwoKICAgIHJvb3QgL3Zhci93d3cvaHRtbDsKICAgIGluZGV4IGluZGV4LnBocCBpbmRleC5odG1sOwoKICAgIGNsaWVudF9tYXhfYm9keV9zaXplIDEwRzsKICAgIGNsaWVudF9ib2R5X3RpbWVvdXQgMzYwMHM7CiAgICBmYXN0Y2dpX2J1ZmZlcnMgNjQgNEs7CgogICAgYWRkX2hlYWRlciBTdHJpY3QtVHJhbnNwb3J0LVNlY3VyaXR5ICJtYXgtYWdlPTE1NzY4MDAwOyBpbmNsdWRlU3ViRG9tYWlucyIgYWx3YXlzOwogICAgYWRkX2hlYWRlciBYLUNvbnRlbnQtVHlwZS1PcHRpb25zIG5vc25pZmYgYWx3YXlzOwogICAgYWRkX2hlYWRlciBYLUZyYW1lLU9wdGlvbnMgU0FNRU9SSUdJTiBhbHdheXM7CiAgICBhZGRfaGVhZGVyIFgtWFNTLVByb3RlY3Rpb24gIjE7IG1vZGU9YmxvY2siIGFsd2F5czsKICAgIGFkZF9oZWFkZXIgWC1Sb2JvdHMtVGFnIG5vbmUgYWx3YXlzOwogICAgYWRkX2hlYWRlciBYLURvd25sb2FkLU9wdGlvbnMgbm9vcGVuIGFsd2F5czsKICAgIGFkZF9oZWFkZXIgWC1QZXJtaXR0ZWQtQ3Jvc3MtRG9tYWluLVBvbGljaWVzIG5vbmUgYWx3YXlzOwogICAgYWRkX2hlYWRlciBSZWZlcnJlci1Qb2xpY3kgbm8tcmVmZXJyZXIgYWx3YXlzOwoKICAgIGxvY2F0aW9uID0gL3JvYm90cy50eHQgeyBhY2Nlc3NfbG9nIG9mZjsgbG9nX25vdF9mb3VuZCBvZmY7IH0KICAgIGxvY2F0aW9uID0gLy53ZWxsLWtub3duL2NhcmRkYXYgeyByZXR1cm4gMzAxICRzY2hlbWU6Ly8kaHR0cF9ob3N0L3JlbW90ZS5waHAvZGF2OyB9CiAgICBsb2NhdGlvbiA9IC8ud2VsbC1rbm93bi9jYWxkYXYgIHsgcmV0dXJuIDMwMSAkc2NoZW1lOi8vJGh0dHBfaG9zdC9yZW1vdGUucGhwL2RhdjsgfQogICAgbG9jYXRpb24gPSAvLndlbGwta25vd24vd2ViZmluZ2VyIHsgcmV0dXJuIDMwMSAkc2NoZW1lOi8vJGh0dHBfaG9zdC9pbmRleC5waHAvLndlbGwta25vd24vd2ViZmluZ2VyOyB9CiAgICBsb2NhdGlvbiA9IC8ud2VsbC1rbm93bi9ub2RlaW5mbyAgeyByZXR1cm4gMzAxICRzY2hlbWU6Ly8kaHR0cF9ob3N0L2luZGV4LnBocC8ud2VsbC1rbm93bi9ub2RlaW5mbzsgfQoKICAgIGxvY2F0aW9uIH4gXi8oPzpidWlsZHx0ZXN0c3xjb25maWd8bGlifDNyZHBhcnR5fHRlbXBsYXRlc3xkYXRhKS8geyBkZW55IGFsbDsgfQogICAgbG9jYXRpb24gfiBeLyg/OlwufGF1dG90ZXN0fG9jY3xpc3N1ZXxpbmRpZXxkYl98Y29uc29sZSkgeyBkZW55IGFsbDsgfQoKICAgICMgRlBNIHN0YXR1cy9waW5nIOKAlCB1c2VkIG9ubHkgYnkgdXBkYXRlIGF1dG9tYXRpb24ncyBhY3Rpdml0eSBjaGVjaywKICAgICMgaW52b2tlZCB2aWEgYGRvY2tlciBjb21wb3NlIGV4ZWNgIGZyb20gaW5zaWRlIHRoaXMgY29udGFpbmVyIChsb2NhbGhvc3QpLgogICAgIyBOZXZlciByZWFjaGFibGUgZnJvbSBvdXRzaWRlLgogICAgbG9jYXRpb24gfiBeLyhzdGF0dXN8cGluZykkIHsKICAgICAgICBhY2Nlc3NfbG9nIG9mZjsKICAgICAgICBhbGxvdyAxMjcuMC4wLjE7CiAgICAgICAgZGVueSBhbGw7CiAgICAgICAgaW5jbHVkZSBmYXN0Y2dpX3BhcmFtczsKICAgICAgICBmYXN0Y2dpX3Bhc3MgYXBwOjkwMDA7CiAgICAgICAgZmFzdGNnaV9wYXJhbSBTQ1JJUFRfRklMRU5BTUUgJGRvY3VtZW50X3Jvb3QkZmFzdGNnaV9zY3JpcHRfbmFtZTsKICAgIH0KCiAgICBsb2NhdGlvbiAvIHsKICAgICAgICB0cnlfZmlsZXMgJHVyaSAkdXJpLyAvaW5kZXgucGhwJHJlcXVlc3RfdXJpOwogICAgfQoKICAgIGxvY2F0aW9uIH4gXC5waHAoPzokfC8pIHsKICAgICAgICBmYXN0Y2dpX3NwbGl0X3BhdGhfaW5mbyBeKC4rXC5waHApKC8uKikkOwogICAgICAgIGZhc3RjZ2lfcGFzcyBhcHA6OTAwMDsKICAgICAgICBmYXN0Y2dpX2luZGV4IGluZGV4LnBocDsKICAgICAgICBpbmNsdWRlIGZhc3RjZ2lfcGFyYW1zOwogICAgICAgIGZhc3RjZ2lfcGFyYW0gU0NSSVBUX0ZJTEVOQU1FICRkb2N1bWVudF9yb290JGZhc3RjZ2lfc2NyaXB0X25hbWU7CiAgICAgICAgZmFzdGNnaV9wYXJhbSBQQVRIX0lORk8gJGZhc3RjZ2lfcGF0aF9pbmZvOwogICAgICAgIGZhc3RjZ2lfcGFyYW0gSFRUUFMgJG5jX2h0dHBzOwogICAgICAgIGZhc3RjZ2lfcGFyYW0gSFRUUF9YX0ZPUldBUkRFRF9GT1IgJHByb3h5X2FkZF94X2ZvcndhcmRlZF9mb3I7CiAgICAgICAgZmFzdGNnaV9wYXJhbSBIVFRQX1hfRk9SV0FSREVEX1BST1RPICRuY19mb3J3YXJkZWRfcHJvdG87CiAgICAgICAgZmFzdGNnaV9yZWFkX3RpbWVvdXQgMzYwMDsKICAgIH0KCiAgICBsb2NhdGlvbiB+IFwubWpzJCB7CiAgICAgICAgdHlwZXMgeyB0ZXh0L2phdmFzY3JpcHQgbWpzOyB9CiAgICAgICAgZGVmYXVsdF90eXBlIHRleHQvamF2YXNjcmlwdDsKICAgICAgICB0cnlfZmlsZXMgJHVyaSAvaW5kZXgucGhwJHJlcXVlc3RfdXJpOwogICAgICAgIGV4cGlyZXMgNk07CiAgICAgICAgYWNjZXNzX2xvZyBvZmY7CiAgICB9CgogICAgbG9jYXRpb24gfiBcLig/OmNzc3xqc3xzdmd8Z2lmfHBuZ3xqcGd8aWNvfHdvZmYyPykkIHsKICAgICAgICB0cnlfZmlsZXMgJHVyaSAvaW5kZXgucGhwJHJlcXVlc3RfdXJpOwogICAgICAgIGV4cGlyZXMgNk07CiAgICAgICAgYWNjZXNzX2xvZyBvZmY7CiAgICB9Cn0K" | base64 -d > ~/nextcloud/nginx/nextcloud.conf && cd ~/nextcloud && docker compose exec web nginx -t && docker compose exec web nginx -s reload && docker compose exec -T app wget -S -qO- --timeout=5 http://10.0.0.50:8080/.well-known/webfinger 2>&1 | head -10
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
