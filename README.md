# Self-Hosted Supabase with Authelia and Caddy
This setup is for a self-hosted Supabase instance, with Authelia providing authentication for Supabase Studio, replacing Kong's basic auth. It also uses Caddy as a reverse proxy and for TLS certificates, and works on localhost.

## Key Components

- **Supabase** : Open-source backend as a service.
- **Authelia** : Self-hosted single sign-on and two-factor authentication for securing Supabase Studio.
- **Kong** : API gateway used for reverse proxying.
- **Caddy** : Used for TLS certificates and as a reverse proxy.


## Localhost Setup
The setup can be run on localhost for local development. Update the domains to whatever you would like.

- **supabase.app.localhost** : Access Supabase services.
- **auth.app.localhost** : Authelia authentication endpoint.

## Configuration

### 1. Caddyfile

The following configuration sets up Caddy to forward authentication to Authelia and reverse proxy to Kong:

```caddyfile
supabase.app.localhost {
    forward_auth authelia:9091 {
        uri /api/verify?rd=https://auth.app.localhost
        copy_headers Remote-User Remote-Groups Remote-Name Remote-Email
    }
    reverse_proxy kong:8000
}

auth.app.localhost {
    reverse_proxy authelia:9091
}

}
```
### 2. kong.yml Modifications
In the volumes/api/kong.yml file, the following lines are commented to disable basic authentication for Supabase Studio:

```yaml

# - name: basic-auth
#   config:
#     hide_credentials: true
```
### 3. Docker Compose Additions
Add the following services to your docker-compose.yml:

```yaml

  authelia:
    container_name: authelia
    image: authelia/authelia
    restart: unless-stopped
    expose:
      - 9091
    volumes:
      - ./authelia/config:/config
    environment:
      TZ: 'Asia/Kolkata'

  caddy:
    image: caddy
    container_name: caddy
    hostname: caddy
    restart: unless-stopped
    env_file: .env
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - ./caddy_config:/config
      - ./caddy_data:/data
```
# How It Works
- **Caddy** : Handles TLS termination and reverse proxying to Kong and other services.
- **Authelia** : Secures access to Supabase Studio by forwarding authentication requests through /api/verify. It uses headers like Remote-User and Remote-Email to manage authentication.
- **Kong** : API gateway with routes to services such as Supabase, with the basic auth plugin disabled for the Studio.
  
# Usage
- Clone supabase repository and set up the required Docker services ```https://supabase.com/docs/guides/self-hosting/docker```.
- Add Authelia and Caddy services to ```docker-compose.yml``` file
- Update the necessary environment variables.
- Run the stack using ```docker-compose up -d```.

