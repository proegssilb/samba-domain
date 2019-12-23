# Samba Active Directory Domain Controller for Docker

Based on [Fmstrat's work](https://github.com/Fmstrat/samba-domain), which is extensively documented, and seems equipped to handle a wide variety of scenarious. This version adds repository-level automation and support for ARM. Increased compatibility with Docker Swarm and/or Kubernetes is the primary future goal.

Most of the documentation in this README is pulled from the original repo.

Any help testing would be appreciated, as I do not have resources to test beyond my particular setup.

## Environment variables for quick start

* `DOMAIN` defaults to `CORP.EXAMPLE.COM` and should be set to your domain
* `DOMAINPASS` should be set to your administrator password, be it existing or new. This can be removed from the environment after the first setup run.
* `HOSTIP` can be set to the IP you want to advertise.
* `JOIN` defaults to `false` and means the container will provision a new domain. Set this to `true` to join an existing domain.
* `JOINSITE` is optional and can be set to a site name when joining a domain, otherwise the default site will be used.
* `DNSFORWARDER` is optional and if an IP such as `192.168.0.1` is supplied will forward all DNS requests samba can't resolve to that DNS server
* `INSECURELDAP` defaults to `false`. When set to true, it removes the secure LDAP requirement. While this is not recommended for production it is required for some LDAP tools. You can remove it later from the smb.conf file stored in the config directory.
* `MULTISITE` defaults to `false` and tells the container to connect to an OpenVPN site via an ovpn file with no password. For instance, if you have two locations where you run your domain controllers, they need to be able to interact. The VPN allows them to do that.
* `NOCOMPLEXITY` defaults to `false`. When set to `true` it removes password complexity requirements including `complexity, history-length, min-pwd-age, max-pwd-age`

## Volumes for quick start

* `/etc/localtime:/etc/localtime:ro` - Sets the timezone to match the host
* `/data/docker/containers/samba/data/:/var/lib/samba` - Stores samba data so the container can be moved to another host if required.
* `/data/docker/containers/samba/config/samba:/etc/samba/external` - Stores the smb.conf so the container can be mored or updates can be easily made.
* `/data/docker/containers/samba/config/openvpn/docker.ovpn:/docker.ovpn` - Optional for connecting to another site via openvpn.
* `/data/docker/containers/samba/config/openvpn/credentials:/credentials` - Optional for connecting to another site via openvpn that requires a username/password. The format for this file should be two lines, with the username on the first, and the password on the second. Also, make sure your ovpn file contains `auth-user-pass /credentials`

## Downloading and building

You know where the clone button is. As of 2019, cross-platform building requires [enabling experimental features](https://stackoverflow.com/questions/57937733/how-to-enable-experimental-docker-cli-features).

This repo publishes to `proegssilb/samba-domain` on [Docker Hub](https://hub.docker.com/repository/docker/proegssilb/samba-domain).

## Setting things up for the container

To set things up you will first want a new IP on your host machine so that ports don't conflict. A domain controller needs a lot of ports, and will likely conflict with things like dnsmasq. The below commands will do this, and set up some required folders.

```bash
ifconfig eno1:1 192.168.3.222 netmask 255.255.255.0 up
mkdir -p /data/docker/containers/samba/data
mkdir -p /data/docker/containers/samba/config/samba
```

If you plan on using a multi-site VPN, also run:

```bash
mkdir -p /data/docker/containers/samba/config/openvpn
cp /path/to/my/ovpn/MYSITE.ovpn /data/docker/containers/samba/config/openvpn/docker.ovpn
```

## Things to keep in mind

* In some cases on Windows clients, you would join with the domain of CORP, but when entering the computer domain you must enter CORP.EXAMPLE.COM. This seems to be the case when using most any samba based DC.
* Make sure your client's DNS is using the DC, or that your mail DNS is relaying for the domain
* Ensure client's are using corp.example.com as the search suffix
* If you're using a VPN, pay close attention to routes. You don't want to force all traffic through the VPN

## Enabling file sharing

This repo doesn't support file sharing in the same container as the DC.

We're dealing with containers. The Samba Team doesn't encourage file sharing on the same machine as a domain controller, it's easy enough to spin up a second container. If you want to do this, consult [Fmstrat's README](https://github.com/Fmstrat/samba-domain) and the official Samba wiki.

## Keeping things updated

The container is stateless, so you can do a `docker rmi samba-domain` and then restart the container to get the latest published build. However, [Fmstrat's README](https://github.com/Fmstrat/samba-domain) provides some scripts that can reduce the amount of time you spend downloading/building containers.

## Examples with docker run

Keep in mind, for all examples replace `proegssilb/samba-domain` with `samba-domain` if you build your own from GitHub.

Start a new domain, and forward non-resolvable queries to the main DNS server

* Local site is `192.168.3.0`
* Local DC (this one) hostname is `LOCALDC` using the host IP of `192.168.3.222`
* Local main DNS is running on `192.168.3.1`

```bash
docker run -t -i \
 -e "DOMAIN=CORP.EXAMPLE.COM" \
 -e "DOMAINPASS=ThisIsMyAdminPassword" \
 -e "DNSFORWARDER=192.168.3.1" \
 -e "HOSTIP=192.168.3.222" \
 -p 192.168.3.222:53:53 \
 -p 192.168.3.222:53:53/udp \
 -p 192.168.3.222:88:88 \
 -p 192.168.3.222:88:88/udp \
 -p 192.168.3.222:135:135 \
 -p 192.168.3.222:137-138:137-138/udp \
 -p 192.168.3.222:139:139 \
 -p 192.168.3.222:389:389 \
 -p 192.168.3.222:389:389/udp \
 -p 192.168.3.222:445:445 \
 -p 192.168.3.222:464:464 \
 -p 192.168.3.222:464:464/udp \
 -p 192.168.3.222:636:636 \
 -p 192.168.3.222:1024-1044:1024-1044 \
 -p 192.168.3.222:3268-3269:3268-3269 \
 -v /etc/localtime:/etc/localtime:ro \
 -v /data/docker/containers/samba/data/:/var/lib/samba \
 -v /data/docker/containers/samba/config/samba:/etc/samba/external \
 --dns-search corp.example.com \
 --dns 192.168.3.222 \
 --dns 192.168.3.1 \
 --add-host localdc.corp.example.com:192.168.3.222 \
 -h localdc \
 --name samba \
 --privileged \
 proegssilb/samba-domain
```

Join an existing domain, and forward non-resolvable queries to the main DNS server

* Local site is `192.168.3.0`
* Local DC (this one) hostname is `LOCALDC` using the host IP of `192.168.3.222`
* Local existing DC is running DNS and has IP of `192.168.3.201`
* Local main DNS is running on `192.168.3.1`

```bash
docker run -t -i \
 -e "DOMAIN=CORP.EXAMPLE.COM" \
 -e "DOMAINPASS=ThisIsMyAdminPassword" \
 -e "JOIN=true" \
 -e "DNSFORWARDER=192.168.3.1" \
 -e "HOSTIP=192.168.3.222" \
 -p 192.168.3.222:53:53 \
 -p 192.168.3.222:53:53/udp \
 -p 192.168.3.222:88:88 \
 -p 192.168.3.222:88:88/udp \
 -p 192.168.3.222:135:135 \
 -p 192.168.3.222:137-138:137-138/udp \
 -p 192.168.3.222:139:139 \
 -p 192.168.3.222:389:389 \
 -p 192.168.3.222:389:389/udp \
 -p 192.168.3.222:445:445 \
 -p 192.168.3.222:464:464 \
 -p 192.168.3.222:464:464/udp \
 -p 192.168.3.222:636:636 \
 -p 192.168.3.222:1024-1044:1024-1044 \
 -p 192.168.3.222:3268-3269:3268-3269 \
 -v /etc/localtime:/etc/localtime:ro \
 -v /data/docker/containers/samba/data/:/var/lib/samba \
 -v /data/docker/containers/samba/config/samba:/etc/samba/external \
 --dns-search corp.example.com \
 --dns 192.168.3.222 \
 --dns 192.168.3.1 \
 --dns 192.168.3.201 \
 --add-host localdc.corp.example.com:192.168.3.222 \
 -h localdc \
 --name samba \
 --privileged \
 proegssilb/samba-domain
```

Join an existing domain, forward DNS, remove security features, and connect to a remote site via openvpn

* Local site is `192.168.3.0`
* Local DC (this one) hostname is `LOCALDC` using the host IP of `192.168.3.222`
* Local existing DC is running DNS and has IP of `192.168.3.201`
* Local main DNS is running on `192.168.3.1`
* Remote site is `192.168.6.0`
* Remote DC hostname is `REMOTEDC` with IP of `192.168.6.222` (notice the DNS and host entries)

```bash
docker run -t -i \
 -e "DOMAIN=CORP.EXAMPLE.COM" \
 -e "DOMAINPASS=ThisIsMyAdminPassword" \
 -e "JOIN=true" \
 -e "DNSFORWARDER=192.168.3.1" \
 -e "MULTISITE=true" \
 -e "NOCOMPLEXITY=true" \
 -e "INSECURELDAP=true" \
 -e "HOSTIP=192.168.3.222" \
 -p 192.168.3.222:53:53 \
 -p 192.168.3.222:53:53/udp \
 -p 192.168.3.222:88:88 \
 -p 192.168.3.222:88:88/udp \
 -p 192.168.3.222:135:135 \
 -p 192.168.3.222:137-138:137-138/udp \
 -p 192.168.3.222:139:139 \
 -p 192.168.3.222:389:389 \
 -p 192.168.3.222:389:389/udp \
 -p 192.168.3.222:445:445 \
 -p 192.168.3.222:464:464 \
 -p 192.168.3.222:464:464/udp \
 -p 192.168.3.222:636:636 \
 -p 192.168.3.222:1024-1044:1024-1044 \
 -p 192.168.3.222:3268-3269:3268-3269 \
 -v /etc/localtime:/etc/localtime:ro \
 -v /data/docker/containers/samba/data/:/var/lib/samba \
 -v /data/docker/containers/samba/config/samba:/etc/samba/external \
 -v /data/docker/containers/samba/config/openvpn/docker.ovpn:/docker.ovpn \
 -v /data/docker/containers/samba/config/openvpn/credentials:/credentials \
 --dns-search corp.example.com \
 --dns 192.168.3.222 \
 --dns 192.168.3.1 \
 --dns 192.168.6.222 \
 --dns 192.168.3.201 \
 --add-host localdc.corp.example.com:192.168.3.222 \
 --add-host remotedc.corp.example.com:192.168.6.222 \
 --add-host remotedc:192.168.6.222 \
 -h localdc \
 --name samba \
 --privileged \
 --cap-add=NET_ADMIN --device /dev/net/tun \
 proegssilb/samba-domain
```

## Examples with docker compose

**THIS SECTION NEEDS TO BE UPDATED FOR UPDATED DOCKER COMPOSE VERSIONS**

Keep in mind for all examples `DOMAINPASS` can be removed after the first run.

Start a new domain, and forward non-resolvable queries to the main DNS server

* Local site is `192.168.3.0`
* Local DC (this one) hostname is `LOCALDC` using the host IP of `192.168.3.222`
* Local main DNS is running on `192.168.3.1`

```yaml
version: '2'

networks:
  extnet:
    external: true

services:

# ----------- samba begin ----------- #

  samba:
    image: proegssilb/samba-domain
    container_name: samba
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /data/docker/containers/samba/data/:/var/lib/samba
      - /data/docker/containers/samba/config/samba:/etc/samba/external
    environment:
      - DOMAIN=CORP.EXAMPLE.COM
      - DOMAINPASS=ThisIsMyAdminPassword
      - DNSFORWARDER=192.168.3.1
      - HOSTIP=192.168.3.222
    networks:
      - extnet
    ports:
      - 192.168.3.222:53:53
      - 192.168.3.222:53:53/udp
      - 192.168.3.222:88:88
      - 192.168.3.222:88:88/udp
      - 192.168.3.222:135:135
      - 192.168.3.222:137-138:137-138/udp
      - 192.168.3.222:139:139
      - 192.168.3.222:389:389
      - 192.168.3.222:389:389/udp
      - 192.168.3.222:445:445
      - 192.168.3.222:464:464
      - 192.168.3.222:464:464/udp
      - 192.168.3.222:636:636
      - 192.168.3.222:1024-1044:1024-1044
      - 192.168.3.222:3268-3269:3268-3269
    dns_search:
      - corp.example.com
    dns:
      - 192.168.3.222
      - 192.168.3.1
    extra_hosts:
      - localdc.corp.example.com:192.168.3.222
    hostname: localdc
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun
    privileged: true
    restart: always

# ----------- samba end ----------- #
```

Join an existing domain, and forward non-resolvable queries to the main DNS server

* Local site is `192.168.3.0`
* Local DC (this one) hostname is `LOCALDC` using the host IP of `192.168.3.222`
* Local existing DC is running DNS and has IP of `192.168.3.201`
* Local main DNS is running on `192.168.3.1`

```yaml
version: '2'

networks:
  extnet:
    external: true

services:

# ----------- samba begin ----------- #

  samba:
    image: proegssilb/samba-domain
    container_name: samba
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /data/docker/containers/samba/data/:/var/lib/samba
      - /data/docker/containers/samba/config/samba:/etc/samba/external
    environment:
      - DOMAIN=CORP.EXAMPLE.COM
      - DOMAINPASS=ThisIsMyAdminPassword
      - JOIN=true
      - DNSFORWARDER=192.168.3.1
      - HOSTIP=192.168.3.222
    networks:
      - extnet
    ports:
      - 192.168.3.222:53:53
      - 192.168.3.222:53:53/udp
      - 192.168.3.222:88:88
      - 192.168.3.222:88:88/udp
      - 192.168.3.222:135:135
      - 192.168.3.222:137-138:137-138/udp
      - 192.168.3.222:139:139
      - 192.168.3.222:389:389
      - 192.168.3.222:389:389/udp
      - 192.168.3.222:445:445
      - 192.168.3.222:464:464
      - 192.168.3.222:464:464/udp
      - 192.168.3.222:636:636
      - 192.168.3.222:1024-1044:1024-1044
      - 192.168.3.222:3268-3269:3268-3269
    dns_search:
      - corp.example.com
    dns:
      - 192.168.3.222
      - 192.168.3.1
      - 192.168.3.201
    extra_hosts:
      - localdc.corp.example.com:192.168.3.222
    hostname: localdc
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun
    privileged: true
    restart: always

# ----------- samba end ----------- #
```

Join an existing domain, forward DNS, remove security features, and connect to a remote site via openvpn

* Local site is `192.168.3.0`
* Local DC (this one) hostname is `LOCALDC` using the host IP of `192.168.3.222`
* Local existing DC is running DNS and has IP of `192.168.3.201`
* Local main DNS is running on `192.168.3.1`
* Remote site is `192.168.6.0`
* Remote DC hostname is `REMOTEDC` with IP of `192.168.6.222` (notice the DNS and host entries)

```yaml
version: '2'

networks:
  extnet:
    external: true

services:

# ----------- samba begin ----------- #

  samba:
    image: proegssilb/samba-domain
    container_name: samba
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /data/docker/containers/samba/data/:/var/lib/samba
      - /data/docker/containers/samba/config/samba:/etc/samba/external
      - /data/docker/containers/samba/config/openvpn/docker.ovpn:/docker.ovpn
      - /data/docker/containers/samba/config/openvpn/credentials:/credentials
    environment:
      - DOMAIN=CORP.EXAMPLE.COM
      - DOMAINPASS=ThisIsMyAdminPassword
      - JOIN=true
      - DNSFORWARDER=192.168.3.1
      - MULTISITE=true
      - NOCOMPLEXITY=true
      - INSECURELDAP=true
      - HOSTIP=192.168.3.222
    networks:
      - extnet
    ports:
      - 192.168.3.222:53:53
      - 192.168.3.222:53:53/udp
      - 192.168.3.222:88:88
      - 192.168.3.222:88:88/udp
      - 192.168.3.222:135:135
      - 192.168.3.222:137-138:137-138/udp
      - 192.168.3.222:139:139
      - 192.168.3.222:389:389
      - 192.168.3.222:389:389/udp
      - 192.168.3.222:445:445
      - 192.168.3.222:464:464
      - 192.168.3.222:464:464/udp
      - 192.168.3.222:636:636
      - 192.168.3.222:1024-1044:1024-1044
      - 192.168.3.222:3268-3269:3268-3269
    dns_search:
      - corp.example.com
    dns:
      - 192.168.3.222
      - 192.168.3.1
      - 192.168.6.222
      - 192.168.3.201
    extra_hosts:
      - localdc.corp.example.com:192.168.3.222
      - remotedc.corp.example.com:192.168.6.222
      - remotedc:192.168.6.222
    hostname: localdc
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun
    privileged: true
    restart: always

# ----------- samba end ----------- #
```

## Joining the domain with Ubuntu

For joining the domain with any client, everything should work just as you would expect if the active directory server was Windows based. For Ubuntu, there are many guides availble for joining, but to make things easier you can find an easily configurable script for joining your domain here: <https://raw.githubusercontent.com/Fmstrat/samba-domain/master/ubuntu-join-domain.sh>

## Troubleshooting

The most common issue is when running multi-site and seeing the below DNS replication error when checking replication with `docker exec samba samba-tool drs showrepl`

```log
CN=Schema,CN=Configuration,DC=corp,DC=example,DC=local
        Default-First-Site-Name\REMOTEDC via RPC
                DSA object GUID: faf297a8-6cd3-4162-b204-1945e4ed5569
                Last attempt @ Thu Jun 29 10:49:45 2017 EDT failed, result 2 (WERR_BADFILE)
                4 consecutive failure(s).
                Last success @ NTTIME(0)
```

This has nothing to do with docker, but does happen in samba setups. The key is to put the GUID host entry into the start script for docker, and restart the container. For instance, if you saw the above error, Add this to you docker command:

```bash
--add-host faf297a8-6cd3-4162-b204-1945e4ed5569._msdcs.corp.example.com:192.168.6.222 \
```

Where `192.168.6.222` is the IP of `REMOTEDC`. You could also do this in `extra_hosts` in docker-compose.
