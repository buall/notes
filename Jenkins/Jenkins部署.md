## Docker运行Jenkins

```bash
docker run \
    -u root \
    -d \
    --name jenkins \
    --restart always \
    -p 18080:8080 \
    -p 50000:50000 \
    -v jenkins-data:/var/jenkins_home \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -e TZ="Asia/Shanghai"  \
    -e JENKINS_OPTS="--prefix=/jenkins" \
    jenkinsci/blueocean
```

## Nginx反向代理Jenkins

```coffeescript
map $http_upgrade $connection_upgrade {
  default upgrade;
  '' close;
}

server {
    listen       80;
    server_name  localhost;

    # Jenkins
    location /jenkins/ {
        sendfile off;
        proxy_pass         http://127.0.0.1:18080;
        proxy_redirect     default;
        proxy_http_version 1.1;
		
        # Required for Jenkins websocket agents
        proxy_set_header   Connection        $connection_upgrade;
        proxy_set_header   Upgrade           $http_upgrade;

        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_max_temp_file_size 0;
		
        #this is the maximum upload size
        client_max_body_size       10m;
        client_body_buffer_size    128k;

        proxy_connect_timeout      90;
        proxy_send_timeout         90;
        proxy_read_timeout         90;
        proxy_buffering            off;
        proxy_request_buffering    off; # Required for HTTP CLI commands
        proxy_set_header Connection ""; # Clear for keepalive
    }
}
```



