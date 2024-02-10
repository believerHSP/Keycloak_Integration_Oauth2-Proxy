# Keycloak_Integration_Oauth2-Proxy  < https://github.com/oauth2-proxy/oauth2-proxy?tab=readme-ov-file#installation >

1. Spin up a Keycloak container with Postgres as DB:
   
#!/bin/bash
##Create a pod with name keycloak and expose port 9080
podman pod create --name keycloaktst --publish 9070:8080 --publish 5434:5432

##Create a postgres container, give it the desired environment variables, attach it to the created pod
podman run -dt \
    --pod keycloaktst \
    --name keycloak-postgrestst \
    -e POSTGRES_DB=keycloak \
    -e POSTGRES_USER=keycloak \
    -e POSTGRES_PASSWORD=redhat \
    -e PGDATA=/var/lib/postgresql/data/pgdata \
    -v /home/lenovo/scratch/mytest/pvol/:/var/lib/postgresql/data \
    postgres

##Create a keycloak container, give it the desired environment variables, importantly DB_VENDOR, attach it to the created pod
podman run -dt \
        --pod keycloaktst \
        --name keycloak-apptst \
        -e KEYCLOAK_ADMIN=admin \
        -e KEYCLOAK_ADMIN_PASSWORD=redhat \
        -e DB_VENDOR=postgres \
        -e DB_ADDR=localhost:5432 \
        -e DB_USER=keycloak \
        -e DB_PASSWORD=redhat \
        -e PROXY_ADDRESS_FORWARDING='true' \
              quay.io/keycloak/keycloak:23.0.3 \
              start-dev
              
2. Configuring Keycloak for OAuth2-proxy: < https://oauth2-proxy.github.io/oauth2-proxy/configuration/oauth_provider#keycloak-oidc-auth-provider >
   Once running, login into keycloak => <IP_ADDRESS:9080> and click on administration console.
   1. Creating the client:
      -------------------
      Create a new OIDC client in your Keycloak realm by navigating to:
         Clients -> Create client
         Client Type 'OpenID Connect'
         Client ID <your client's id>, please complete the remaining fields as appropriate and click Next.
         Client authentication 'On'
      Authentication flow
         Standard flow 'selected'
         Direct access grants 'deselect'
      Save the configuration.
      Settings / Access settings:
         Valid redirect URIs https://internal.yourcompany.com/oauth2/callback
         Save the configuration.
      Under the Credentials tab you will now be able to locate <your client's secret>.
      
   2. Configure a dedicated audience mapper for your client by navigating to Clients -> <your client's id> -> Client scopes.
      --------
      
   
4. Spin up an Oauth2-Proxy container:
   
    #!/bin/bash
    podman run -d \
               --name oauth2-proxy \
               -p 4180:4180 \
               -v /home/lenovo/scratch/podman/oauth2_proxy/oauth2-proxy.cfg:/etc/oauth2-proxy.cfg \
                quay.io/oauth2-proxy/oauth2-proxy --config=/etc/oauth2-proxy.cfg --provider=keycloak-oidc --oidc-issuer-url=http://192.168.122.1:9080/auth/realms/staging-auth

   #podman run -d --name oauth2-proxy -p 4100:4180 -v /home/lenovo/scratch/mytest/oath2proxy/oauth2-proxy.cfg:/etc/oauth2-proxy.cfg quay.io/oauth2-proxy/oauth2-proxy -- 
    config=/etc/oauth2-proxy.cfg --provider=keycloak-oidc --oidc-issuer-url=http://192.168.122.1:9070/realms/filestash

5. OAuth2-proxy configuration-Modifications:
## OAuth2 Proxy Config File
## https://github.com/oauth2-proxy/oauth2-proxy

## <addr>:<port> to listen on for HTTP/HTTPS clients
http_address = ":4180"
# https_address = ":443"

## Are we running behind a reverse proxy? Will not accept headers like X-Real-Ip unless this is set.
# reverse_proxy = true

## TLS Settings
# tls_cert_file = ""
# tls_key_file = ""

## the OAuth Redirect URL.
# defaults to the "https://" + requested host header + "/oauth2/callback"
# redirect_url = "https://internalapp.yourcompany.com/oauth2/callback"
redirect_url = "http://192.168.122.1:4170/oauth2/callback"


## the http url(s) of the upstream endpoint. If multiple, routing is based on path
upstreams = [
     "http://192.168.122.1:8334/"
]

## Logging configuration
logging_filename = "/tmp/oauth2.log"
logging_max_size = 100
logging_max_age = 7
logging_local_time = true
logging_compress = false
standard_logging = true
standard_logging_format = "[{{.Timestamp}}] [{{.File}}] {{.Message}}"
request_logging = true
request_logging_format = "{{.Client}} - {{.Username}} [{{.Timestamp}}] {{.Host}} {{.RequestMethod}} {{.Upstream}} {{.RequestURI}} {{.Protocol}} {{.UserAgent}} {{.StatusCode}} {{.ResponseSize}} {{.RequestDuration}}"
auth_logging = true
auth_logging_format = "{{.Client}} - {{.Username}} [{{.Timestamp}}] [{{.Status}}] {{.Message}}"

## pass HTTP Basic Auth, X-Forwarded-User and X-Forwarded-Email information to upstream
# pass_basic_auth = true
# pass_user_headers = true
## pass the request Host Header to upstream
## when disabled the upstream Host is used as the Host Header
# pass_host_header = true

## Email Domains to allow authentication for (this authorizes any email on this domain)
## for more granular authorization use `authenticated_emails_file`
## To authorize any email addresses use "*"
email_domains = [
     "*"
]
## The OAuth Client ID, Secret
# client_id = "123456.apps.googleusercontent.com"
# client_secret = ""
client_id= "staging-auth"
client_secret = "Z1P7LhzoQqkzhJij4Hd85Uce6aCeg84d"

## Pass OAuth Access token to upstream via "X-Forwarded-Access-Token"
pass_access_token = false

## Authenticated Email Addresses File (one email per line)
# authenticated_emails_file = ""

## Htpasswd File (optional)
## Additionally authenticate against a htpasswd file. Entries must be created with "htpasswd -B" for bcrypt encryption
## enabling exposes a username/login signin form
# htpasswd_file = ""

## bypass authentication for requests that match the method & path. Format: method=path_regex OR path_regex alone for all methods
# skip_auth_routes = [
#   "GET=^/probe",
#   "^/metrics"
# ]

## mark paths as API routes to get HTTP Status code 401 instead of redirect to login page
# api_routes = [
#   "^/api
# ]

## Templates
## optional directory with custom sign_in.html and error.html
# custom_templates_dir = ""

## skip SSL checking for HTTPS requests
# ssl_insecure_skip_verify = false


## Cookie Settings
## Name     - the cookie name
## Secret   - the seed string for secure cookies; should be 16, 24, or 32 bytes
##            for use with an AES cipher when cookie_refresh or pass_access_token
##            is set
## Domain   - (optional) cookie domain to force cookies to (ie: .yourcompany.com)
## Expire   - (duration) expire timeframe for cookie
## Refresh  - (duration) refresh the cookie when duration has elapsed after cookie was initially set.
##            Should be less than cookie_expire; set to 0 to disable.
##            On refresh, OAuth token is re-validated.
## Secure   - secure cookies are only sent by the browser of a HTTPS connection (recommended)
## HttpOnly - httponly cookies are not readable by javascript (recommended)
cookie_name = "_oauth2_proxy"
cookie_secret = "wYTBIWPQ_1C-hL-WxVu_i1z8t19ZxiDQY5AADDqtI7Y="
#cookie_domains = "localhost"
cookie_expire = "168h"
cookie_refresh = "0"
cookie_secure = false
cookie_httponly = true
insecure_oidc_allow_unverified_email = "true"

# Changes 
#cookie_csrf_per_request=true
#cookie_csrf_expire=5m

5. Restart the OAuth2-proxy container.
