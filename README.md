# Docker-dev

> **⚠️ Warning**
>
> These are config examples and should be used at your own risk.

## Treafik
1. `docker context use docker-dev`
1. `brew install openssl`
1. `openssl genrsa -out conf/tls/root_ca.key 8192`

### Generate ROOT CA
1. `cp conf/tls/openssl.conf.example conf/tls/openssl.conf`
1. Generate private key for root CA: `openssl genrsa -out conf/tls/root_ca.key 8192`
1. Generate certificate for root CA: `COMMON_NAME="example.local" /usr/local/opt/openssl/bin/openssl req -new -x509 -days 3650 -sha512 -extensions v3_ca -key conf/tls/root_ca.key -out conf/tls/root_ca.crt -config conf/tls/openssl.conf`
1. Verify root CA: `openssl x509 -in conf/tls/root_ca.crt -text -noout | head -n 10`
```shell
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            4e:ba:7b:69:c4:14:44:39:f8:73:7c:06:b1:10:aa:ba:d2:8e:00:08
        Signature Algorithm: sha512WithRSAEncryption
        Issuer: C = US, ST = NY, L = Buffalo, O = Example, CN = example.local Root CA
        Validity
            Not Before: Dec 17 07:08:11 2023 GMT
            Not After : Dec 14 07:08:11 2033 GMT
            ...
            ...
```

### Generate intermediate
1. Generate private key for intermediate certificate: `openssl genrsa -out conf/tls/example_local_int.key 4096`
1. Generate certificate signing request (CSR) for intermediate: `COMMON_NAME="example.local Intermediate Certificate" /usr/local/opt/openssl/bin/openssl req -new -sha512 -extensions v3_intermediate_ca -key conf/tls/example_local_int.key -out conf/tls/example_local_int.csr -config conf/tls/openssl.conf`
1. Generate certificate for intermediate with CSR: `COMMON_NAME="example.local Intermediate Certificate" /usr/local/opt/openssl/bin/openssl x509 -req -days 2555 -sha512 -extensions v3_intermediate_ca -in conf/tls/example_local_int.csr -CA conf/tls/root_ca.crt -CAkey conf/tls/root_ca.key -CAcreateserial -out conf/tls/example_local_int.crt -extfile conf/tls/openssl.conf`
```shell
Certificate request self-signature ok
subject=C = US, ST = NY, L = Buffalo, O = Example, CN = example.local Intermediate Certificate Root CA
```

### Generate leaf cert
1. Generate private key for leaf certificate: `openssl genrsa -out conf/tls/traefik_example_local.key 4096`
1. Generate certificate signing request (CSR) for leaf: `COMMON_NAME="*.example.local" /usr/local/opt/openssl/bin/openssl req -new -sha512 -extensions v3_req -key conf/tls/traefik_example_local.key -out conf/tls/traefik_example_local.csr -config conf/tls/openssl.conf`
1. Generate certificate for leaf with CSR:`COMMON_NAME="*.example.local" /usr/local/opt/openssl/bin/openssl x509 -req -days 365 -sha512 -extensions v3_req -in conf/tls/traefik_example_local.csr -CA conf/tls/example_local_int.crt -CAkey conf/tls/example_local_int.key -CAcreateserial -out conf/tls/traefik_example_local.crt -extfile conf/tls/openssl.conf`
```shell
Certificate request self-signature ok
subject=C = US, ST = NY, L = Buffalo, O = Example, CN = *.example.local Root CA
```
1. Append intermediate cert to leaf for cert chain: `cat conf/tls/example_local_int.crt >> conf/tls/traefik_example_local.crt`
1. Append root CA to leaf for cert chain: `cat conf/tls/root_ca.crt >> conf/tls/traefik_example_local.crt`

### Spin up stack
1. `cat conf/traefik/dyn-v3.yaml | docker config create traefik-dyn -`
1. `cat conf/traefik/traefik.yml | docker config create traefik-conf -`
1. `cat conf/tls/traefik_example_local.key | docker secret create traefik-ssl-key -`
1. `cat conf/tls/traefik_example_local.crt  | docker secret create traefik-ssl-crt -`
1. `docker stack deploy -c docker-compose-swarm-traefik.yml traefik`
1. `docker service logs -f traefik_traefik`
1. `docker service logs -f traefik_traefik`
```shell
traefik_traefik.1.p8luwj9ayl3t@docker-dev    | time="2023-12-17T07:53:32Z" level=info msg="Configuration loaded from file: /etc/traefik/traefik.yml"
traefik_traefik.1.p8luwj9ayl3t@docker-dev    | time="2023-12-17T07:53:32Z" level=info msg="Traefik version 2.8.0 built on 2022-06-29T15:43:58Z"
traefik_traefik.1.p8luwj9ayl3t@docker-dev    | time="2023-12-17T07:53:32Z" level=info msg="\nStats collection is disabled.\nHelp us improve Traefik by turning this feature on :)\nMore details on: https://doc.traefik.io/traefik/contributing/data-collection/\n"
traefik_traefik.1.p8luwj9ayl3t@docker-dev    | time="2023-12-17T07:53:32Z" level=warning msg="Traefik Pilot is deprecated and will be removed soon. Please check our Blog for migration instructions later this year."
traefik_traefik.1.p8luwj9ayl3t@docker-dev    | time="2023-12-17T07:53:32Z" level=info msg="Starting provider aggregator aggregator.ProviderAggregator"
traefik_traefik.1.p8luwj9ayl3t@docker-dev    | time="2023-12-17T07:53:32Z" level=info msg="Starting provider *file.Provider"
traefik_traefik.1.p8luwj9ayl3t@docker-dev    | time="2023-12-17T07:53:32Z" level=info msg="Starting provider *traefik.Provider"
traefik_traefik.1.p8luwj9ayl3t@docker-dev    | time="2023-12-17T07:53:32Z" level=info msg="Starting provider *docker.Provider"
traefik_traefik.1.p8luwj9ayl3t@docker-dev    | time="2023-12-17T07:53:32Z" level=info msg="Starting provider *acme.ChallengeTLSALPN"
...
```shell
1. Verify Traefik is proxying traffic: `curl --resolve whoami.<base_domain>:443:<docker IP addr> https://whoami.<base_domain> --verbose -k`
```shell
...
Hostname: d44b284316f1
IP: 127.0.0.1
IP: 10.0.5.6
IP: 172.18.0.6
RemoteAddr: 10.0.5.3:48740
GET / HTTP/1.1
Host: whoami.example.local
...
```
1. Open web browser to `https://traefik.<base_domain>/dashboard/#/`

## Vault
### Setup FreeIPA
https://holdmybeersecurity.com/2021/02/16/my-development-server-for-vault/

### Create Docker secrets
1. `cat conf/tls/root_ca.key | docker secret create vault-example-local-rootCA-key -`
1. `cat conf/tls/root_ca.crt | docker secret create vault-example-local-rootCA-cert -`
1. `cat conf/tls/example_local_int.key | docker secret create vault-example-local-intCA-key -`
1. `cat conf/tls/example_local_int.crt | docker secret create vault-example-local-intCA-cert -`
1. `echo "ldaps://freeipa.<base_domain>" | docker secret create vault-example-local-ldap-bind-url -`
1. `echo "vault" | docker secret create vault-example-local-ldap-bind-username -`
1. `echo "<password>" | docker secret create vault-example-local-ldap-bind-password -`

### Create Docker configs
1. `cat conf/vault/consul-config-swarm.hcl | docker config create vault-example-local-consul-config -`
1. `cat conf/vault/vault-config-swarm.hcl | docker config create vault-example-local-config -`
1. `cat conf/vault/ldap.ldif | docker config create  vault-example-local-ldap-auth-config -`
1. `cat conf/vault/admin-user-policy.hcl| docker config create vault-example-local-ldap-admin-policy -`
1. `cat conf/vault/vault_entrypoint_setup.sh | docker config create vault-example-local-entrypoint-script -`

### Spin up stack
1. `docker stack deploy -c docker-compose-swarm-vault.yml vault`
1. `docker service logs -f vault_vault`

## Cyberchef
1. `docker stack deploy -c docker-compose-swarm-cyberchef.yml cyberchef`

## References
### Cyberchef
* [DockerHub - remnux/cyberchef](https://hub.docker.com/r/remnux/cyberchef/tags)

### Traefik
* [Recommended TLS Ciphers for Traefik](https://stackoverflow.com/questions/52128979/recommended-tls-ciphers-for-traefik)
* [traefik 2.7, intermediate config](https://ssl-config.mozilla.org/#server=traefik&version=2.7&config=intermediate&guideline=5.6)
* [Gitlab (docker) behind traefik v2](https://community.traefik.io/t/gitlab-docker-behind-traefik-v2/2268)
* [traefik](https://hub.docker.com/_/traefik?tab=tags)
* []()
* []()
* []()
