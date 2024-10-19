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

## Clone supabase repository and set up the required Docker services 
```https://supabase.com/docs/guides/self-hosting/docker```.

## Configuration

### 1. Caddyfile

The following configuration sets up Caddy to forward authentication to Authelia and reverse proxy to Kong:

```caddyfile
supabase.app.localhost {
    forward_auth authelia:9091 {
        uri /api/authz/forward-auth?authelia_url=https://auth.app.localhost
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
Add the following services to your docker-compose.yml: (We just updated default_redirection_url,access_control and session in this file )

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
### 4. Authelia Configuration
Add the following services to your authelia/config/configuration.yml:

```yaml

  ## The theme to display: light, dark, grey, auto.
theme: light

## The secret used to generate JWT tokens when validating user identity by email confirmation. JWT Secret can also be
## set using a secret: https://www.authelia.com/c/secrets
jwt_secret: a_very_important_secret

default_redirection_url: https://auth.app.localhost # confusing haxxors
default_2fa_method: ""

server:
  host: 0.0.0.0
  port: 9091
  path: ""
  enable_pprof: false
  enable_expvars: false
  disable_healthcheck: false
  read_buffer_size: 4096
  write_buffer_size: 4096

  ## Server headers configuration/customization.
  headers:
    ## The CSP Template. Read the docs.
    csp_template: ""

  ## Authelia by default doesn't accept TLS communication on the server port. This section overrides this behaviour.
  tls:
    key: ""
    certificate: ""
    client_certificates: []

log:
  level: debug

totp:
  issuer: app.localhost #your authelia top-level domain
  period: 30
  digits: 6
  algorithm: sha1
  skew: 1

authentication_backend:
  file:
    path: /config/users_database.yml
    watch: false
    search:
      email: false
      case_insensitive: false
    password:
      algorithm: argon2
      argon2:
        variant: argon2id
        iterations: 3
        memory: 65536
        parallelism: 4
        key_length: 32
        salt_length: 16
  ## Password Reset Options.
  password_reset:
    ## Disable both the HTML element and the API for reset password functionality.
    disable: false
    ## External reset password url that redirects the user to an external reset portal. This disables the internal reset
    ## functionality.
    custom_url: ""
  refresh_interval: 5m


password_policy:
  ## The standard policy allows you to tune individual settings manually.
  standard:
    enabled: false
    min_length: 8
    max_length: 0
    require_uppercase: true
    require_lowercase: true
    require_number: true
    require_special: true

  ## zxcvbn is a well known and used password strength algorithm. It does not have tunable settings.
  zxcvbn:
    enabled: false
    min_score: 3


access_control:
  ## Default policy can either be 'bypass', 'one_factor', 'two_factor' or 'deny'. It is the policy applied to any
  ## resource if there is no policy to be applied to the user.
  default_policy: deny
  rules:
    - domain:
        - "auth.app.localhost"
      policy: bypass
    - domain: "supabase.app.localhost"
      policy: one_factor

session:
  ## The name of the session cookie.
  name: authelia_session
  domain: app.localhost
  ## https://www.authelia.com/c/session#same_site
  same_site: lax

  ## The secret to encrypt the session data. 
  ## This is only used with Redis / Redis Sentinel.
  secret: insecure_session_secret
  ## The time before the cookie expires and the session is destroyed if remember me IS NOT selected.
  expiration: 1h
  ## The inactivity time before the session is reset. If expiration is set to 1h, and this is set to 5m, if the user
  ## does not select the remember me option their session will get destroyed after 1h, or after 5m since the last time
  ## Authelia detected user activity.
  inactivity: 5m
  ## The time before the cookie expires and the session is destroyed if remember me IS selected.
  ## Value of -1 disables remember me.
  remember_me_duration: 1M

# security measures against hackers
regulation:
  max_retries: 3
  find_time: 2m
  ban_time: 30m

storage:
  encryption_key: a_very_important_secret
  local:
    path: /config/db.sqlite3

notifier:
  disable_startup_check: false
  ## File System (Notification Provider)
  filesystem:
    filename: /config/notification.txt
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
- Default credentials for Authelia are **username:**```authelia``` and **password:**```authelia```
- Make sure to change the username password in the file /authelia/config/users_database.yml.
  ```
  users:
  authelia:
    disabled: false
    displayname: "Test User"
    password: "$argon2id$v=19$m=32768,t=1,p=8$eUhVT1dQa082YVk2VUhDMQ$E8QI4jHbUBt3EdsU1NFDu4Bq5jObKNx7nBKSn1EYQxk"  # Password is 'authelia'
    email: authelia@authelia.com
    groups:
      - admins
      - dev
  ```
- Run this command to generate a hashed password for authelia ``` docker run -it authelia/authelia:latest authelia crypto hash generate argon2```.

