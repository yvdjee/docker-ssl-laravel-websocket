# docker-ssl-laravel-websocket

### Setup containers
```
cd apps
git clone https://github.com/yvdjee/laravel-websocket-nginx-ssl.git web
docker-compose build
docker-compose up -d
```

### Setup laravel
```
docker-compose exec web composer install
cd apps/web
cp .env.example .env
docker-compose exec web php artisan migrate
docker-compose exec web php artisan key:generate
docker-compose exec web chmod -R 777 storage
```

### Display container's IP
```
docker ps -q | xargs -n 1 docker inspect --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}} {{ .Name }}' | sed 's/ \// /'
```

### volumes/proxy/templates/nginx.tmpl
`nano volumes/proxy/templates/nginx.tmpl`

172.18.0.4 = web container's docker ip (replace it)
```
location /ws {
                proxy_pass http://172.18.0.4:6001;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection 'upgrade';
                proxy_set_header X-Forwarded-For $remote_addr;
        }
```
Restart `docker-compose restart nginx`

### resources/js/bootstrap.js
`nano apps/web/resources/js/bootstrap.js`

```
window.Echo = new Echo({
    broadcaster: 'pusher',
    key: process.env.MIX_PUSHER_APP_KEY,
    wsHost: 'yourdomain.com',
    wssPort: 6001,
    disableStats: true,
    scheme: 'https',
    encrypted: true,
    enabledTransports: ['ws', 'wss']
});
```

### config/broadcasting.php
`nano apps/web/config/broadcasting.php`

172.18.0.4 = web container's docker ip (replace it)
```
'pusher' => [
            'driver' => 'pusher',
            'key' => env('PUSHER_APP_KEY'),
            'secret' => env('PUSHER_APP_SECRET'),
            'app_id' => env('PUSHER_APP_ID'),
            'options' => [
                'cluster' => env('PUSHER_APP_CLUSTER'),
                'encrypted' => true,
                'host' => '172.18.0.4',
                'port' => 6001,
                'scheme' => 'https',
                'curl_options' => [
                    CURLOPT_SSL_VERIFYHOST => 0,
                    CURLOPT_SSL_VERIFYPEER => 0,
                ]
            ],
        ],
```
### config/websockets.php
Copy and paste to web root directory `apps/web`
```
volumes/proxy/certs/yourdomain.com.crt
volumes/proxy/certs/yourdomain.com.key
```
```
'ssl' => [
        'local_cert' => base_path('yourdomain.com.crt'),
        'local_pk' => base_path('yourdomain.com.key'),
        'passphrase' => null
    ],
```
### Run websocket server
`docker-compose exec web php artisan websocket:serve`
