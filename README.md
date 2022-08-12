[![Build Status](https://drone.kiwi-labs.net/api/badges/Diesel-Net/openldap/status.svg)](https://drone.kiwi-labs.net/Diesel-Net/openldap)

# openldap
OpenLDAP directory service on docker swarm


## TLS Terminating Proxy
Many hours were spent trying to get Traefik to proxy plaintext ldap (port 389) traffic to the container, whilst terminating TLS. I simply wanted to leverage the ACME features of Traefik to automate my certificate renewals, but never quite got anything working. 

I had traefik entrypoint set to 636 (firewall port on VM open as well) and even `openssl s_client -showcerts ldap.dev.diesel.net:636` was showing `verification OK`. For some reason it would not play nicely with this container, and I cannot figure out quite why. Not sure if it has something to with OpenSSL vs GnuTLS as the underlying ssl ibrary for the openldap container, cipher/encryption algorithm being used, or maybe an incompatibility that Traefik still needs to workout.. Went down several rabbit holes haha.. 

For what it's worth, here were the traefik labels that I came up with, along with traefik command line arguments. Maybe someone can figure it out.
```bash
# Traefik configuration (cli)

--entryPoints.ldap.address=:389
--entryPoints.ldaps.address=:636
```


```bash
# Docker container labels

- traefik.enable=true

# service (ldap, plaintext, port 389)
- traefik.tcp.services.ldap.loadbalancer.server.port=389

# ldap router
- traefik.tcp.routers.ldap.rule=HostSNI(`*`)
- traefik.tcp.routers.ldap.entrypoints=ldap
- traefik.tcp.routers.ldap.service=ldap@docker

# ldaps router
- traefik.tcp.routers.ldaps.rule=HostSNI(`{{ domain }}`)
- traefik.tcp.routers.ldaps.entrypoints=ldaps
- traefik.tcp.routers.ldaps.tls.certresolver=step-ca
- traefik.tcp.routers.ldaps.service=ldap@docker
```


All in all, I could not get Traefik v2.4.11 to play nicely with openldap. In addition, I do not believe that _starttls_ will ever be possible while using a tls terminating proxy, due to the nature of it starting over a non-tls connection, then "upgrading" to a secure connection. So I would recommend to just feed the certificates to openldap and let it do the work. You can still use something like [certbot](https://certbot.eff.org/) with [cron](https://en.wikipedia.org/wiki/Cron) to fully automate certificate renewals.

## Debugging
For official OpenLDAP docs on TLS configuration, visit [this link](https://www.openldap.org/doc/admin24/guide.html#Using%20TLS).

ldapsearch against the running ldap container.
```bash
docker exec -it \
$(docker ps -q -f name=ldap_development_main) \
ldapsearch -x -d1 -H ldaps://ldap.dev.diesel.net -b dc=diesel,dc=net -D "cn=admin,dc=diesel,dc=net" -W
```

ldapsearch, using a standalone container, preferabbly on a remote machine that has been [bootstrapped with root-ca](https://github.com/Diesel-Net/ansible-role-ubuntu/blob/94663769c1d999ee7ba397538d13a68577ae50bc/tasks/main.yaml#L46). All the commands below are slight variations to test ldap, ldaps, and starts appropriatley. You can always add the `-d1` flag to add more verbose debug output.
```bash
# ldap
docker run -it \
-v /etc/ssl/certs/:/etc/ssl/certs/ \
--entrypoint ldapsearch \
osixia/openldap:1.5.0 \
-x -H ldap://ldap.dev.diesel.net -b dc=diesel,dc=net -D cn=admin,dc=diesel,dc=net -W

# ldaps
docker run -it \
-v /etc/ssl/certs/:/etc/ssl/certs/ \
--entrypoint ldapsearch \
osixia/openldap:1.5.0 \
-x -H ldaps://ldap.dev.diesel.net -b dc=diesel,dc=net -D cn=admin,dc=diesel,dc=net -W

# starttls
docker run -it \
-v /etc/ssl/certs/:/etc/ssl/certs/ \
--entrypoint ldapsearch \
osixia/openldap:1.5.0 \
-x -H ldap://ldap.dev.diesel.net -b dc=diesel,dc=net -D cn=admin,dc=diesel,dc=net -W -ZZ
```

```bash
openssl s_client -showcerts ldap.dev.diesel.net:636
```

## Dependencies
- ansible-core 2.13+


## Installing Dependencies
```bash
ansible-galaxy role install -r .ansible/roles/requirements.yaml -p .ansible/roles --force
ansible-galaxy collection install -r .ansible/roles/requirements.yaml --force
```

## Deploy
Right now each environment is defined as an independent Virtual Machine (single-node swarm leaders)
```bash
ansible-playbook .ansible/deploy.yaml -i .ansible/inventory/dev/hosts
```

