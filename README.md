# nginxplus-apigateway
<!-- ABOUT THE PROJECT -->
## About The Project

API gateways can be used for both monolithic and microservices-based apps. API gateways perform several key functions:

* Authenticating the requesters making API calls
* Routing requests to appropriate backends
* Applying rate limits to prevent overloading of your systems
* Handling errors and exceptions
* Terminating SSL traffic

**This is the base config:** apigw.conf
```nginx
upstream backend-api {
    server 18.143.131.133:85;
}

server {
   listen 80;
   server_name apigw.mikelabs.online;
   location / {
        proxy_http_version 1.1;
        proxy_set_header   "Connection" "";
        proxy_pass          http://backend-api;
    }

}
```
We will implement rate limiting based on source-ip address, together with error handling

**This is the config with rate limiting and error handling:** apigw-ratelimiting.conf
```nginx
upstream backend-api {
    server 18.143.131.133:85;
}

limit_req_zone $binary_remote_addr zone=ratelimit:20m rate=1r/s;
limit_req_status 429;

server {
   listen 80;
   server_name apigw.mikelabs.online;
   location / {
        limit_req zone=ratelimit;
        proxy_http_version 1.1;
        proxy_set_header   "Connection" "";
        proxy_pass          http://backend-api;
        error_page 429 /error429.json;
    }
    location = /error429.json {
        internal;
        return 429 '{"status": "error", "message": "429 Too many requests !"}';
    }
}
```

**This is the config with TLS termination:** apigw-tls.conf
```nginx
upstream backend-api {
    server 18.143.131.133:85;
}

limit_req_zone $binary_remote_addr zone=ratelimit:20m rate=1r/s;
limit_req_status 429;

server {
   listen 443 ssl;
   server_name apigw.mikelabs.online;
   ssl_certificate      /etc/ssl/certs/api.example.com.crt;
   ssl_certificate_key  /etc/ssl/private/api.example.com.key;
   ssl_session_cache    shared:SSL:10m;
   ssl_session_timeout  5m;
   ssl_ciphers          HIGH:!aNULL:!MD5;
   ssl_protocols        TLSv1.2 TLSv1.3;

   location / {
        limit_req zone=ratelimit;
        proxy_http_version 1.1;
        proxy_set_header   "Connection" "";
        proxy_pass          http://backend-api;
        error_page 429 /error429.json;
    }
    location = /error429.json {
        internal;
        return 429 '{"status": "error", "message": "429 Too many requests !"}';
    }
}
```

**This is the config with authentication via apikey:** apigw-authentication.conf
```nginx
upstream backend-api {
    server 18.143.131.133:85;
}

map $http_apikey $api_client_name {
    default "";
    "7B5zIqmRGXmrJTFmKa99vcit" "client_one";
    "QzVV6y1EmQFbbxOfRCwyJs35" "client_two";
    "mGcjH8Fv6U9y3BVF9H3Ypb9T" "client_three";
}

limit_req_zone $binary_remote_addr zone=ratelimit:20m rate=1r/s;
limit_req_status 429;

server {
   listen 443 ssl;
   server_name apigw.mikelabs.online;
   ssl_certificate      /etc/ssl/certs/api.example.com.crt;
   ssl_certificate_key  /etc/ssl/private/api.example.com.key;
   ssl_session_cache    shared:SSL:10m;
   ssl_session_timeout  5m;
   ssl_ciphers          HIGH:!aNULL:!MD5;
   ssl_protocols        TLSv1.2 TLSv1.3;

   location / {
        limit_req zone=ratelimit;
        proxy_http_version 1.1;
        proxy_set_header   "Connection" "";
        auth_request /_validate_apikey;
        proxy_pass          http://backend-api;
        error_page 429 /error429.json;
        error_page 401 /error401.json;
        error_page 403 /error403.json;
    }
    location = /_validate_apikey {
        internal;
        if ($http_apikey = "") {
            return 401; # Unauthorized
        }
        if ($api_client_name = "") {
            return 403; # Forbidden
        }
        return 204; # OK (no content)
    }
    location = /error429.json {
        internal;
        return 429 '{"status": "error", "message": "429 Too many requests !"}';
    }
    location = /error401.json {
        internal;
        return 401 '{"status": "error", "message": "401 customized Authorization Required !"}';
    }
    location = /error403.json {
        internal;
        return 403 '{"status": "error", "message": "403 Customized Forbidden !"}';
    }
}
```

**NGINX PLUS: Enable API Dashboard**

```nginx
upstream backend-api {
    server 18.143.131.133:85;
    zone backend-api 32k;
}

server {
   listen 80;
   server_name apigw.mikelabs.online;
   status_zone apigw.mikelabs.online;
   location / {
        status_zone /;
        proxy_http_version 1.1;
        proxy_set_header   "Connection" "";
        proxy_pass          http://backend-api;
    }

}
```

**NGINX PLUS: App Protect WAF**

```nginx
#add this module on /etc/nginx/nginx.conf
load_module modules/ngx_http_app_protect_module.so;
```

create another server block for the dashboard using dashboard.conf 

```nginx
server {
    listen 8080;
    location /api {
        api write=on;
        # directives limiting access to the API
    }

    location = /dashboard.html {
        root   /usr/share/nginx/html;
    }

    # Redirect requests made to the pre-NGINX Plus API dashboard
    location = /status.html {
        return 301 /dashboard.html;
    }
}
```
