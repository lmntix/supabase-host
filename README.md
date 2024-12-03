This setup is for a self-hosted Supabase instance, with Authelia providing authentication for Supabase Studio, replacing Kong's basic auth. It also uses Caddy as a reverse proxy and for TLS certificates, and works on localhost as well. Lets go step by step:

**For reference** : https://github.com/lmntix/supabase-host

# Step 1
Follow the official Supabase Documentation to get the files from github repository.

Docs URL: https://supabase.com/docs/guides/self-hosting/docker
```
# Get the code
git clone --depth 1 https://github.com/supabase/supabase

# Go to the docker folder
cd supabase/docker

# Copy the fake env vars
cp .env.example .env
```
# Step 2
Update the .env files with below:
```
# Your Supabase API 
API_EXTERNAL_URL: supabase.app.localhost

# Your Supabase Dashboard URL
SUPABASE_PUBLIC_URL: studio.app.localhost
```
# Step 3
Navigate to volumes/api/kong.yml and comment out the last 3 steps as shown below if you do not want the basic auth. I will comment it out since I dont want the basic auth and will be using Authelia instead.
```
  ## Protected Dashboard - catch all remaining routes
  - name: dashboard
    _comment: 'Studio: /* -> http://studio:3000/*'
    url: http://studio:3000/
    routes:
      - name: dashboard-all
        strip_path: true
        paths:
          - /
    plugins:
      - name: cors
      # - name: basic-auth
      #   config:
      #     hide_credentials: true
```
# Step 4
Navigate back to supabase/docker, if not already there.

Add below Caddy and Authelia services in the docker-compose.yml file
```
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
# Step 5
Create a Caddyfile in the same directory. This will be used by the caddy container for TLS.

(This is going to forward us to authelia portal when trying to access the supabase dashboard. Only after authentication we would be able to access the dashboard.)

Update the Caddyfile with domains for below services:

- Supabase Dashboard - studio.app.localhost
- Supabase API - supabase.app.localhost
- Authelia Auth Portal - auth.app.localhost
  
```
studio.app.localhost {
    forward_auth authelia:9091 {
        uri /api/authz/forward-auth?authelia_url=https://auth.app.localhost
        copy_headers Remote-User Remote-Groups Remote-Name Remote-Email
    }
    reverse_proxy kong:8000
}

auth.app.localhost {
    reverse_proxy authelia:9091
}

supabase.app.localhost {
    reverse_proxy kong:8000
}
```
# Step 6
Run ``` docker-compose up -d ``` This will create necessary files for Authelia which we now need to update.

# Step 7
Navigate to authelia/config/confihuration.yml
Replace the content with below.

We basically update at 3 places with our domains.

- default_redirection_url: https://auth.app.localhost
- issuer: app.localhost
- access_control domains

```
  ## The theme to display: light, dark, grey, auto.
theme: light

## The secret used to generate JWT tokens when validating user identity by email confirmation. JWT Secret can also be
## set using a secret: 
jwt_secret: a_very_important_secret

default_redirection_url: https://auth.app.localhost # confusing haxxors
default_2fa_method: ""

server:
  host: 
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
    - domain: "studio.app.localhost"
      policy: one_factor

session:
  ## The name of the session cookie.
  name: authelia_session
  domain: app.localhost
  ## 
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
    filename: /config/notification.txthttps://www.authelia.com/c/secretshttps://auth.app.localhost0.0.0.0https://www.authelia.com/c/session#same_site

```
# Step 8
Go to users_database.yml file. Default username and password present there would be authelia and authelia. You can update the list of username passwords there for those who would be able to access the supabase studio.

Use below code in terminal to generate a hashed password to use here.

``` docker run -it authelia/authelia:latest authelia crypto hash generate argon2 ```
```
users:
  username:
    disabled: false
    displayname: "User Full Name"
    password: "$argon2id$v=19$m=65536,t=3,p=4$tMQZmtuSDygs/D9dereergtbShb2inyrrtgeDt/AH96tJcKZO3MWXRetkPMZdhhWo"  
    email: lmntix@example.com
    groups:
      - admins
      - dev
 ```     
# Step 9
We are done!! Go back to the folder supabase/docker which has our docker compose file.

``` docker-compose up -d --force-recreate ```

If you want to logout, you can simply go back to auth.app.localhost and click on logout.


# Important things to note after installation.

- If you want to connect to db directly from external client, supabase now provides supavisor to enable you for the same at Port 5432/6543 for pooler/transaction mode.

``` postgresql://postgres.[POOLER_TENANT_ID]:[POSTGRES_PASSWORD]@[YOUR_DOMAIN]:6543/postgres ```

- In order to create a new JWT token you can run below command in the terminal

``` node -e "console.log(require('crypto').randomBytes(32).toString('hex'))" ```

- When you create a new JWT Token for Supabase, you would need to create a new ANON_KEY as well as new SERVICE_ROLE key as well.

In order to generate those there are 2 sources.
```
- https://supabase.com/docs/guides/self-hosting/docker#generate-api-keys 
- https://jwt.io/
```

- There have been several issues raised that I saw about storage not working in supabase dashboard. That basically had to do with incorrect SERVICE_ROLE key being set. So if thats the issue you are facing as well, you can simple go to jwt.io and use it to create a proper key and then use it in env variables.

It uses below objects:

ANON KEY
```
{
  "role": "anon",
  "iss": "supabase",
  "iat": 1729276200,
  "exp": 1887042600
}
```
SERVICE_ROLE KEY
```
{
  "role": "service_role",
  "iss": "supabase",
  "iat": 1729276200,
  "exp": 1887042600
}
```
- When password is changed like for postgres in .env , It causes an error when started again docker compose up -d 

Solution
You need to do run below commands
```
docker compose down -v
rm -rf volumes/db/data/
docker compose up -d
```


# Happy Coding!!!
