# pgadmin-bug-report

**Describe the bug**

I am not sure if this is a bug or if I have configured something wrong.
Currently I have pgadmin behind an SSL nginx reverse proxy in a docker compose environment.
When I added an SSL certificate, it appears I am getting the following error below!

![image](https://github.com/user-attachments/assets/9cc1cb0e-af2b-401f-9178-e4ab6d2bf34c)

**To Reproduce**

Steps to reproduce the behavior:
1. Go to [https://github.com/iWolf22/pgadmin-bug-report](https://github.com/iWolf22/pgadmin-bug-report)
2. Run the application in Docker
3. You will need to add a `.env` file in the root with `PGADMIN_DEFAULT_EMAIL` and `PGADMIN_DEFAULT_PASSWORD`
4. You will also need a `.env.dhparam.pem`, `.env.server.cer`, and `.env.server.key` which are a `ssl_dhparam`, `ssl_certificate`, and a `ssl_certificate_key` respectively (located in `/utils/nginx`)

 - For reference, `docker-compose.yml`
 ```yaml
services:
    pgadmin:
        container_name: pgadmin
        image: dpage/pgadmin4:8.9
        restart: always
        ports:
            - 5050:80
        environment:
            PGADMIN_DEFAULT_EMAIL: ${PGADMIN_DEFAULT_EMAIL}
            PGADMIN_DEFAULT_PASSWORD: ${PGADMIN_DEFAULT_PASSWORD}
        volumes:
            - pgadmin-data:/var/lib/pgadmin

    nginx:
        container_name: nginx
        image: nginx:1.27
        restart: always
        ports:
            - 80:80
            - 443:443
        volumes:
            - ./utils/nginx/nginx.conf:/etc/nginx/nginx.conf
            - ./utils/nginx/.env.dhparam.pem:/etc/nginx/ssl/.env.dhparam.pem
            - ./utils/nginx/.env.server.cer:/etc/nginx/ssl/.env.server.cer
            - ./utils/nginx/.env.server.key:/etc/nginx/ssl/.env.server.key

volumes:
    pgadmin-data:

```


 - For reference, `nginx.conf`
```nginx
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
    worker_connections 1024;
}


http {

    # http server
    server {

        # Listen on port 80 (http) for incoming traffic
        listen 80 default_server;

        # Server name, will match with any hostnamed used
        server_name _;

        # Redirect to the https version of the given URL
        return 301 https://$host$request_uri;
    }

    # https server
    server {

        # Listen on port 443 (https) for incoming traffic
        listen 443 default ssl;
        listen [::]:443 ssl;
        http2 on;

        # Server name, will match with any hostnamed used
        server_name _;

        # Only return Nginx in server header
        # The less information revealed to potential hackers, the better
        server_tokens off;

        # Turns on SSL and specifies the location of the keys/certificates
        # ssl on;
        ssl_certificate /etc/nginx/ssl/.env.server.cer;
        ssl_certificate_key /etc/nginx/ssl/.env.server.key;

        # Diffie-Hellman (DH) cryptographic algorithm key
        ssl_dhparam /etc/nginx/ssl/.env.dhparam.pem;

        # Use the newest (SSL/TLS) protocols
        # They use strong ecryption algorithms and key lenghts
        ssl_protocols TLSv1.2 TLSv1.3;

        # Compilation of the top cipher suites 2024
        # https://ssl-config.mozilla.org/#server=nginx
        ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-CHACHA20-POLY1305;

        # Directive to tell the server to use its own perfereed ciphers, rather than the clients
        # Perfect Forward Secrecy(PFS) is frequently compromised without this
        ssl_prefer_server_ciphers on;

        # Disables SSL session tickets (which utilize SSL sessions and improve performance)
        ssl_session_tickets off;

        # Enable SSL session caching for improved performance
        ssl_session_timeout 1d;
        ssl_session_cache shared:SSL:10m; # about 40000 sessions

        # By default, the buffer size is 16k, which corresponds to minimal overhead when sending big responses.
        # To minimize Time To First Byte it may be beneficial to use smaller values
        ssl_buffer_size 8k;

        # OCSP stapling
        ssl_stapling on;
        ssl_stapling_verify on;

        # Security headers
        ## X-Content-Type-Options: avoid MIME type sniffing
        add_header X-Content-Type-Options nosniff;
        ## Content-Security-Policy (CSP): Yes
        ## No 'script-src' directive, you need to test it yourself
        add_header Content-Security-Policy "object-src 'none'; base-uri 'none'; require-trusted-types-for 'script'; frame-ancestors 'self';";
        ## The safest CSP, only block your website to be inside an inframe
        add_header Content-Security-Policy "frame-ancestors 'self';";
        ## Strict Transport Security (HSTS): Yes
        add_header Strict-Transport-Security "max-age=31536000; includeSubdomains; preload";

        location /pgadmin {
            proxy_set_header X-Script-Name /pgadmin;
            proxy_set_header X-Scheme $scheme;
            proxy_set_header Host $host;
            proxy_pass http://pgadmin:80;
            proxy_redirect off;
        }
    }
}
```

**Expected behavior**

For pgadmin to work behind a nginx subdirectory with an SSL certificate.

**Error message**

 - DOM Error 1
```
This document requires 'TrustedScriptURL' assignment.
req.load | @ | require.min.js?ver=80900:9
```

 - DOM Error 2
```
Uncaught TypeError: Failed to set the 'src' property on 'HTMLScriptElement': This document requires 'TrustedScriptURL' assignment.
    at req.load (require.min.js?ver=80900:9:3650)
    at Object.load (require.min.js?ver=80900:9:16691)
    at i.load (require.min.js?ver=80900:9:10095)
    at i.fetch (require.min.js?ver=80900:9:9896)
    at i.check (require.min.js?ver=80900:9:11160)
    at i.enable (require.min.js?ver=80900:9:13253)
    at Object.enable (require.min.js?ver=80900:9:15801)
    at i.<anonymous> (require.min.js?ver=80900:9:13110)
    at require.min.js?ver=80900:9:1542
    at each (require.min.js?ver=80900:9:1020)
```

**Desktop (please complete the following information):**
 - OS: Windows 10
 - Mode: Desktop
 - Browser: Chrome 126
