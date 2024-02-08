# Keycloak_Integration_Oauth2-Proxy

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
              
2. Configuring Keycloak for OAuth2-proxy:
   
   
3. Spin up an Oauth2-Proxy container:
   
    #!/bin/bash
    podman run -d \
               --name oauth2-proxy \
               -p 4180:4180 \
               -v /home/lenovo/scratch/podman/oauth2_proxy/oauth2-proxy.cfg:/etc/oauth2-proxy.cfg \
                quay.io/oauth2-proxy/oauth2-proxy --config=/etc/oauth2-proxy.cfg --provider=keycloak-oidc --oidc-issuer-url=http://192.168.122.1:9080/auth/realms/staging-auth

   #podman run -d --name oauth2-proxy -p 4100:4180 -v /home/lenovo/scratch/mytest/oath2proxy/oauth2-proxy.cfg:/etc/oauth2-proxy.cfg quay.io/oauth2-proxy/oauth2-proxy -- 
    config=/etc/oauth2-proxy.cfg --provider=keycloak-oidc --oidc-issuer-url=http://192.168.122.1:9070/realms/filestash

   
