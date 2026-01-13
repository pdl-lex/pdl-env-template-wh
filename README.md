# Test Environment Template

Minimal setup to run isolated environments on the LexoTerm server.

The basic idea is to forward requests to specific subdomains to internal ports via the main 
Caddyfile of the production setup. The requests are then handled by the local Caddyfile.

## Setup

1. Fork this repository
2. Set up a domain in the *production* Caddyfile in [pdl-deployment](https://github.com/pdl-lex/pdl-deployment) with a free port (`808x`):

    ```caddy
    example.lexoterm.de {
        reverse_proxy host.docker.internal:8080
    }
    ```
    (Remember to update and redeploy the infrastructure.)

3. Set the same port in the caddy service of your *local* [compose.yml](compose.yml).
    ```yml
    services:
      caddy:
        image: caddy:latest
        ports:
        - "808x:80"  # Match with port from step 1
        - ...
    ```
    You can keep `80` for the right-hand side.

3. Spin up the test service with `docker compose up -d`


After a moment, the domain should display the *Hello World* test page from html/index.html. From there, update your local Caddyfile and services as desired.