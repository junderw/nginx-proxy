[![Test](https://github.com/nginx-proxy/nginx-proxy/actions/workflows/test.yml/badge.svg)](https://github.com/nginx-proxy/nginx-proxy/actions/workflows/test.yml)
[![GitHub release](https://img.shields.io/github/v/release/nginx-proxy/nginx-proxy)](https://github.com/nginx-proxy/nginx-proxy/releases)
![nginx 1.21.1](https://img.shields.io/badge/nginx-1.21.1-brightgreen.svg)
[![Docker Image Size](https://img.shields.io/docker/image-size/nginxproxy/nginx-proxy?sort=semver)](https://hub.docker.com/r/nginxproxy/nginx-proxy "Click to view the image on Docker Hub")
[![Docker stars](https://img.shields.io/docker/stars/nginxproxy/nginx-proxy.svg)](https://hub.docker.com/r/nginxproxy/nginx-proxy 'DockerHub')
[![Docker pulls](https://img.shields.io/docker/pulls/nginxproxy/nginx-proxy.svg)](https://hub.docker.com/r/nginxproxy/nginx-proxy 'DockerHub')


nginx-proxy sets up a container running nginx and [docker-gen](https://github.com/nginx-proxy/docker-gen). docker-gen generates reverse proxy configs for nginx and reloads nginx when containers are started and stopped.

See [Automated Nginx Reverse Proxy for Docker](http://jasonwilder.com/blog/2014/03/25/automated-nginx-reverse-proxy-for-docker/) for why you might want to use this.

### Usage

To run it:

```console
docker run -d -p 80:80 -v /var/run/docker.sock:/tmp/docker.sock:ro nginxproxy/nginx-proxy
```

Then start any containers you want proxied with an env var `VIRTUAL_HOST=subdomain.youdomain.com`

```console
docker run -e VIRTUAL_HOST=foo.bar.com  ...
```

The containers being proxied must [expose](https://docs.docker.com/engine/reference/run/#expose-incoming-ports) the port to be proxied, either by using the `EXPOSE` directive in their `Dockerfile` or by using the `--expose` flag to `docker run` or `docker create` and be in the same network. By default, if you don't pass the --net flag when your nginx-proxy container is created, it will only be attached to the default bridge network. This means that it will not be able to connect to containers on networks other than bridge.

Provided your DNS is setup to forward foo.bar.com to the host running nginx-proxy, the request will be routed to a container with the `VIRTUAL_HOST` env var set.

Note: providing a port number in `VIRTUAL_HOST` isn't suported, please see [virtual ports](https://github.com/nginx-proxy/nginx-proxy#virtual-ports) or [custom external HTTP/HTTPS ports](https://github.com/nginx-proxy/nginx-proxy#virtual-ports) depending on what you want to achieve.

### Image variants

The nginx-proxy images are available in two flavors.

#### nginxproxy/nginx-proxy:latest

This image uses the debian:buster based nginx image.

```console
docker pull nginxproxy/nginx-proxy:latest
```

#### nginxproxy/nginx-proxy:alpine

This image is based on the nginx:alpine image. Use this image to fully support HTTP/2 (including ALPN required by recent Chrome versions). A valid certificate is required as well (see eg. below "SSL Support using an ACME CA" for more info).

```console
docker pull nginxproxy/nginx-proxy:alpine
```

### Docker Compose

```yaml
version: '2'

services:
  nginx-proxy:
    image: nginxproxy/nginx-proxy
    ports:
      - "80:80"
    volumes:
      - /var/run/docker.sock:/tmp/docker.sock:ro

  whoami:
    image: jwilder/whoami
    expose:
      - "8000"
    environment:
      - VIRTUAL_HOST=whoami.local
      - VIRTUAL_PORT=8000
```

```console
docker-compose up
curl -H "Host: whoami.local" localhost
```

Example output:
```console
I'm 5b129ab83266
```

### IPv6 support

You can activate the IPv6 support for the nginx-proxy container by passing the value `true` to the `ENABLE_IPV6` environment variable:

```console
docker run -d -p 80:80 -e ENABLE_IPV6=true -v /var/run/docker.sock:/tmp/docker.sock:ro nginxproxy/nginx-proxy
```

#### Scoped IPv6 Resolvers

NginX does not support scoped IPv6 resolvers. In [docker-entrypoint.sh](./docker-entrypoint.sh) the resolvers are parsed from resolv.conf, but any scoped IPv6 addreses will be removed. 

#### IPv6 NAT

By default, docker uses IPv6-to-IPv4 NAT. This means all client connections from IPv6 addresses will show docker's internal IPv4 host address. To see true IPv6 client IP addresses, you must [enable IPv6](https://docs.docker.com/config/daemon/ipv6/) and use [ipv6nat](https://github.com/robbertkl/docker-ipv6nat). You must also disable the userland proxy by adding `"userland-proxy": false` to `/etc/docker/daemon.json` and restarting the daemon.

### Multiple Hosts

If you need to support multiple virtual hosts for a container, you can separate each entry with commas.  For example, `foo.bar.com,baz.bar.com,bar.com` and each host will be setup the same.

### Virtual Ports

When your container exposes only one port, nginx-proxy will default to this port, else to port 80.

If you need to specify a different port, you can set a `VIRTUAL_PORT` env var to select a different one. This variable cannot be set to more than one port.

For each host defined into `VIRTUAL_HOST`, the associated virtual port is retrieved by order of precedence:
1. From the `VIRTUAL_PORT` environment variable
1. From the container's exposed port if there is only one
1. From the default port 80 when none of the above methods apply

### Wildcard Hosts

You can also use wildcards at the beginning and the end of host name, like `*.bar.com` or `foo.bar.*`. Or even a regular expression, which can be very useful in conjunction with a wildcard DNS service like [xip.io](http://xip.io), using `~^foo\.bar\..*\.xip\.io` will match `foo.bar.127.0.0.1.xip.io`, `foo.bar.10.0.2.2.xip.io` and all other given IPs. More information about this topic can be found in the nginx documentation about [`server_names`](http://nginx.org/en/docs/http/server_names.html).

### Multiple Networks

With the addition of [overlay networking](https://docs.docker.com/engine/userguide/networking/get-started-overlay/) in Docker 1.9, your `nginx-proxy` container may need to connect to backend containers on multiple networks. By default, if you don't pass the `--net` flag when your `nginx-proxy` container is created, it will only be attached to the default `bridge` network. This means that it will not be able to connect to containers on networks other than `bridge`.

If you want your `nginx-proxy` container to be attached to a different network, you must pass the `--net=my-network` option in your `docker create` or `docker run` command. At the time of this writing, only a single network can be specified at container creation time. To attach to other networks, you can use the `docker network connect` command after your container is created:

```console
docker run -d -p 80:80 -v /var/run/docker.sock:/tmp/docker.sock:ro \
    --name my-nginx-proxy --net my-network nginxproxy/nginx-proxy
docker network connect my-other-network my-nginx-proxy
```

In this example, the `my-nginx-proxy` container will be connected to `my-network` and `my-other-network` and will be able to proxy to other containers attached to those networks.

### Custom external HTTP/HTTPS ports

If you want to use `nginx-proxy` with different external ports that the default ones of `80` for `HTTP` traffic and `443` for `HTTPS` traffic, you'll have to use the environment variable(s) `HTTP_PORT` and/or `HTTPS_PORT` in addition to the changes to the Docker port mapping. If you change the `HTTPS` port, the redirect for `HTTPS` traffic will also be configured to redirect to the custom port. Typical usage, here with the custom ports `1080` and `10443`:

```console
docker run -d -p 1080:1080 -p 10443:10443 -e HTTP_PORT=1080 -e HTTPS_PORT=10443 -v /var/run/docker.sock:/tmp/docker.sock:ro nginxproxy/nginx-proxy
```

### Internet vs. Local Network Access

If you allow traffic from the public internet to access your `nginx-proxy` container, you may want to restrict some containers to the internal network only, so they cannot be accessed from the public internet.  On containers that should be restricted to the internal network, you should set the environment variable `NETWORK_ACCESS=internal`.  By default, the *internal* network is defined as `127.0.0.0/8, 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16`.  To change the list of networks considered internal, mount a file on the `nginx-proxy` at `/etc/nginx/network_internal.conf` with these contents, edited to suit your needs:

```Nginx
# These networks are considered "internal"
allow 127.0.0.0/8;
allow 10.0.0.0/8;
allow 192.168.0.0/16;
allow 172.16.0.0/12;

# Traffic from all other networks will be rejected
deny all;
```

When internal-only access is enabled, external clients will be denied with an `HTTP 403 Forbidden`

> If there is a load-balancer / reverse proxy in front of `nginx-proxy` that hides the client IP (example: AWS Application/Elastic Load Balancer), you will need to use the nginx `realip` module (already installed) to extract the client's IP from the HTTP request headers.  Please see the [nginx realip module configuration](http://nginx.org/en/docs/http/ngx_http_realip_module.html) for more details.  This configuration can be added to a new config file and mounted in `/etc/nginx/conf.d/`.

### SSL Backends

If you would like the reverse proxy to connect to your backend using HTTPS instead of HTTP, set `VIRTUAL_PROTO=https` on the backend container.

> Note: If you use `VIRTUAL_PROTO=https` and your backend container exposes port 80 and 443, `nginx-proxy` will use HTTPS on port 80.  This is almost certainly not what you want, so you should also include `VIRTUAL_PORT=443`.

### uWSGI Backends

If you would like to connect to uWSGI backend, set `VIRTUAL_PROTO=uwsgi` on the backend container. Your backend container should then listen on a port rather than a socket and expose that port.

### FastCGI Backends
 
If you would like to connect to FastCGI backend, set `VIRTUAL_PROTO=fastcgi` on the backend container. Your backend container should then listen on a port rather than a socket and expose that port.
 
### FastCGI File Root Directory

If you use fastcgi,you can set `VIRTUAL_ROOT=xxx`  for your root directory


### Default Host

To set the default host for nginx use the env var `DEFAULT_HOST=foo.bar.com` for example

```console
docker run -d -p 80:80 -e DEFAULT_HOST=foo.bar.com -v /var/run/docker.sock:/tmp/docker.sock:ro nginxproxy/nginx-proxy
```

nginx-proxy will then redirect all requests to a container where `VIRTUAL_HOST` is set to `DEFAULT_HOST`, if they don't match any (other) `VIRTUAL_HOST`. Using the example above requests without matching `VIRTUAL_HOST` will be redirected to a plain nginx instance after running the following command:

```console
docker run -d -e VIRTUAL_HOST=foo.bar.com nginx
```

### Separate Containers

nginx-proxy can also be run as two separate containers using the [jwilder/docker-gen](https://hub.docker.com/r/jwilder/docker-gen) image and the official [nginx](https://registry.hub.docker.com/_/nginx/) image.

You may want to do this to prevent having the docker socket bound to a publicly exposed container service.

You can demo this pattern with docker-compose:

```console
docker-compose --file docker-compose-separate-containers.yml up
curl -H "Host: whoami.local" localhost
```

Example output:
```console
I'm 5b129ab83266
```

To run nginx proxy as a separate container you'll need to have [nginx.tmpl](https://github.com/nginx-proxy/nginx-proxy/blob/main/nginx.tmpl) on your host system.

First start nginx with a volume:


```console
docker run -d -p 80:80 --name nginx -v /tmp/nginx:/etc/nginx/conf.d -t nginx
```

Then start the docker-gen container with the shared volume and template:

```console
docker run --volumes-from nginx \
    -v /var/run/docker.sock:/tmp/docker.sock:ro \
    -v $(pwd):/etc/docker-gen/templates \
    -t jwilder/docker-gen -notify-sighup nginx -watch /etc/docker-gen/templates/nginx.tmpl /etc/nginx/conf.d/default.conf
```

Finally, start your containers with `VIRTUAL_HOST` environment variables.

```console
docker run -e VIRTUAL_HOST=foo.bar.com  ...
```

### SSL Support using an ACME CA

[acme-companion](https://github.com/nginx-proxy/acme-companion) is a lightweight companion container for the nginx-proxy. It allows the automated creation/renewal of SSL certificates using the ACME protocol.

Set `DHPARAM_GENERATION` environment variable to `false` to disabled Diffie-Hellman parameters completely. This will also ignore auto-generation made by `nginx-proxy`. The default value is `true`

```console
docker run -e DHPARAM_GENERATION=false ....
```

### SSL Support

SSL is supported using single host, wildcard and SNI certificates using naming conventions for certificates or optionally specifying a cert name (for SNI) as an environment variable.

To enable SSL:

```console
docker run -d -p 80:80 -p 443:443 -v /path/to/certs:/etc/nginx/certs -v /var/run/docker.sock:/tmp/docker.sock:ro nginxproxy/nginx-proxy
```

The contents of `/path/to/certs` should contain the certificates and private keys for any virtual hosts in use. The certificate and keys should be named after the virtual host with a `.crt` and `.key` extension. For example, a container with `VIRTUAL_HOST=foo.bar.com` should have a `foo.bar.com.crt` and `foo.bar.com.key` file in the certs directory.

If you are running the container in a virtualized environment (Hyper-V, VirtualBox, etc...), /path/to/certs must exist in that environment or be made accessible to that environment. By default, Docker is not able to mount directories on the host machine to containers running in a virtual machine.

#### Diffie-Hellman Groups

Diffie-Hellman groups are enabled by default, with a pregenerated key in `/etc/nginx/dhparam/dhparam.pem`. You can mount a different `dhparam.pem` file at that location to override the default cert. To use custom `dhparam.pem` files per-virtual-host, the files should be named after the virtual host with a `dhparam` suffix and `.pem` extension. For example, a container with `VIRTUAL_HOST=foo.bar.com` should have a `foo.bar.com.dhparam.pem` file in the `/etc/nginx/certs` directory.

> NOTE: If you don't mount a `dhparam.pem` file at `/etc/nginx/dhparam/dhparam.pem`, one will be generated at startup.  Since it can take minutes to generate a new `dhparam.pem`, it is done at low priority in the background. Once generation is complete, the `dhparam.pem` is saved on a persistent volume and nginx is reloaded. This generation process only occurs the first time you start `nginx-proxy`.

> COMPATIBILITY WARNING: The default generated `dhparam.pem` key is 4096 bits for A+ security. Some older clients (like Java 6 and 7) do not support DH keys with over 1024 bits. In order to support these clients, you must either provide your own `dhparam.pem`, or tell `nginx-proxy` to generate a 1024-bit key on startup by passing `-e DHPARAM_BITS=1024`.

In the separate container setup, no pregenerated key will be available and neither the [jwilder/docker-gen](https://hub.docker.com/r/jwilder/docker-gen) image nor the offical [nginx](https://registry.hub.docker.com/_/nginx/) image will generate one. If you still want A+ security in a separate container setup, you'll have to generate a 2048 or 4096 bits DH key file manually and mount it on the nginx container, at `/etc/nginx/dhparam/dhparam.pem`.

#### Wildcard Certificates

Wildcard certificates and keys should be named after the domain name with a `.crt` and `.key` extension. For example `VIRTUAL_HOST=foo.bar.com` would use cert name `bar.com.crt` and `bar.com.key`.

#### SNI

If your certificate(s) supports multiple domain names, you can start a container with `CERT_NAME=<name>` to identify the certificate to be used.  For example, a certificate for `*.foo.com` and `*.bar.com` could be named `shared.crt` and `shared.key`.  A container running with `VIRTUAL_HOST=foo.bar.com` and `CERT_NAME=shared` will then use this shared cert.

#### OCSP Stapling

To enable OCSP Stapling for a domain, `nginx-proxy` looks for a PEM certificate containing the trusted CA certificate chain at `/etc/nginx/certs/<domain>.chain.pem`, where `<domain>` is the domain name in the `VIRTUAL_HOST` directive. The format of this file is a concatenation of the public PEM CA certificates starting with the intermediate CA most near the SSL certificate, down to the root CA. This is often referred to as the "SSL Certificate Chain". If found, this filename is passed to the NGINX [`ssl_trusted_certificate` directive](http://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_trusted_certificate) and OCSP Stapling is enabled.

#### How SSL Support Works

The default SSL cipher configuration is based on the [Mozilla intermediate profile](https://wiki.mozilla.org/Security/Server_Side_TLS#Intermediate_compatibility_.28recommended.29) version 5.0 which should provide compatibility with clients back to Firefox 27, Android 4.4.2, Chrome 31, Edge, IE 11 on Windows 7, Java 8u31, OpenSSL 1.0.1, Opera 20, and Safari 9. Note that the DES-based TLS ciphers were removed for security. The configuration also enables HSTS, PFS, OCSP stapling and SSL session caches. Currently TLS 1.2 and 1.3 are supported.

If you don't require backward compatibility, you can use the [Mozilla modern profile](https://wiki.mozilla.org/Security/Server_Side_TLS#Modern_compatibility) profile instead by including the environment variable `SSL_POLICY=Mozilla-Modern` to the nginx-proxy container or to your container. This profile is compatible with clients back to Firefox 63, Android 10.0, Chrome 70, Edge 75, Java 11, OpenSSL 1.1.1, Opera 57, and Safari 12.1.  Note that this profile is **not** compatible with any version of Internet Explorer.

Other policies available through the `SSL_POLICY` environment variable are [`Mozilla-Old`](https://wiki.mozilla.org/Security/Server_Side_TLS#Old_backward_compatibility) and the [AWS ELB Security Policies](https://docs.aws.amazon.com/elasticloadbalancing/latest/classic/elb-security-policy-table.html) `AWS-TLS-1-2-2017-01`, `AWS-TLS-1-1-2017-01`, `AWS-2016-08`, `AWS-2015-05`, `AWS-2015-03` and `AWS-2015-02`.

Note that the `Mozilla-Old` policy should use a 1024 bits DH key for compatibility but this container generates a 4096 bits key. The [Diffie-Hellman Groups](#diffie-hellman-groups) section details different methods of bypassing this, either globally or per virtual-host.

The default behavior for the proxy when port 80 and 443 are exposed is as follows:

* If a container has a usable cert, port 80 will redirect to 443 for that container so that HTTPS is always preferred when available.
* If the container does not have a usable cert, a 503 will be returned.

Note that in the latter case, a browser may get an connection error as no certificate is available to establish a connection. A self-signed or generic cert named `default.crt` and `default.key` will allow a client browser to make a SSL connection (likely w/ a warning) and subsequently receive a 500.

To serve traffic in both SSL and non-SSL modes without redirecting to SSL, you can include the environment variable `HTTPS_METHOD=noredirect` (the default is `HTTPS_METHOD=redirect`). You can also disable the non-SSL site entirely with `HTTPS_METHOD=nohttp`, or disable the HTTPS site with `HTTPS_METHOD=nohttps`. `HTTPS_METHOD` can be specified on each container for which you want to override the default behavior or on the proxy container to set it globally. If `HTTPS_METHOD=noredirect` is used, Strict Transport Security (HSTS) is disabled to prevent HTTPS users from being redirected by the client. If you cannot get to the HTTP site after changing this setting, your browser has probably cached the HSTS policy and is automatically redirecting you back to HTTPS. You will need to clear your browser's HSTS cache or use an incognito window / different browser.

By default, [HTTP Strict Transport Security (HSTS)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Strict-Transport-Security)  is enabled with `max-age=31536000` for HTTPS sites. You can disable HSTS with the environment variable `HSTS=off` or use a custom HSTS configuration like `HSTS=max-age=31536000; includeSubDomains; preload`. 

*WARNING*: HSTS will force your users to visit the HTTPS version of your site for the `max-age` time - even if they type in `http://` manually.  The only way to get to an HTTP site after receiving an HSTS response is to clear your browser's HSTS cache.

### Basic Authentication Support

In order to be able to secure your virtual host, you have to create a file named as its equivalent VIRTUAL_HOST variable on directory
/etc/nginx/htpasswd/$VIRTUAL_HOST

```console
docker run -d -p 80:80 -p 443:443 \
    -v /path/to/htpasswd:/etc/nginx/htpasswd \
    -v /path/to/certs:/etc/nginx/certs \
    -v /var/run/docker.sock:/tmp/docker.sock:ro \
    nginxproxy/nginx-proxy
```

You'll need apache2-utils on the machine where you plan to create the htpasswd file. Follow these [instructions](http://httpd.apache.org/docs/2.2/programs/htpasswd.html)

### Custom Nginx Configuration

If you need to configure Nginx beyond what is possible using environment variables, you can provide custom configuration files on either a proxy-wide or per-`VIRTUAL_HOST` basis.

#### Replacing default proxy settings

If you want to replace the default proxy settings for the nginx container, add a configuration file at `/etc/nginx/proxy.conf`. A file with the default settings would look like this:

```Nginx
# HTTP 1.1 support
proxy_http_version 1.1;
proxy_buffering off;
proxy_set_header Host $http_host;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection $proxy_connection;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $proxy_x_forwarded_proto;
proxy_set_header X-Forwarded-Ssl $proxy_x_forwarded_ssl;
proxy_set_header X-Forwarded-Port $proxy_x_forwarded_port;

# Mitigate httpoxy attack (see README for details)
proxy_set_header Proxy "";
```

***NOTE***: If you provide this file it will replace the defaults; you may want to check the .tmpl file to make sure you have all of the needed options.

***NOTE***: The default configuration blocks the `Proxy` HTTP request header from being sent to downstream servers.  This prevents attackers from using the so-called [httpoxy attack](http://httpoxy.org).  There is no legitimate reason for a client to send this header, and there are many vulnerable languages / platforms (`CVE-2016-5385`, `CVE-2016-5386`, `CVE-2016-5387`, `CVE-2016-5388`, `CVE-2016-1000109`, `CVE-2016-1000110`, `CERT-VU#797896`).

#### Proxy-wide

To add settings on a proxy-wide basis, add your configuration file under `/etc/nginx/conf.d` using a name ending in `.conf`.

This can be done in a derived image by creating the file in a `RUN` command or by `COPY`ing the file into `conf.d`:

```Dockerfile
FROM nginxproxy/nginx-proxy
RUN { \
      echo 'server_tokens off;'; \
      echo 'client_max_body_size 100m;'; \
    } > /etc/nginx/conf.d/my_proxy.conf
```

Or it can be done by mounting in your custom configuration in your `docker run` command:

```console
docker run -d -p 80:80 -p 443:443 -v /path/to/my_proxy.conf:/etc/nginx/conf.d/my_proxy.conf:ro -v /var/run/docker.sock:/tmp/docker.sock:ro nginxproxy/nginx-proxy
```

#### Per-VIRTUAL_HOST

To add settings on a per-`VIRTUAL_HOST` basis, add your configuration file under `/etc/nginx/vhost.d`. Unlike in the proxy-wide case, which allows multiple config files with any name ending in `.conf`, the per-`VIRTUAL_HOST` file must be named exactly after the `VIRTUAL_HOST`.

In order to allow virtual hosts to be dynamically configured as backends are added and removed, it makes the most sense to mount an external directory as `/etc/nginx/vhost.d` as opposed to using derived images or mounting individual configuration files.

For example, if you have a virtual host named `app.example.com`, you could provide a custom configuration for that host as follows:

```console
docker run -d -p 80:80 -p 443:443 -v /path/to/vhost.d:/etc/nginx/vhost.d:ro -v /var/run/docker.sock:/tmp/docker.sock:ro nginxproxy/nginx-proxy
{ echo 'server_tokens off;'; echo 'client_max_body_size 100m;'; } > /path/to/vhost.d/app.example.com
```

If you are using multiple hostnames for a single container (e.g. `VIRTUAL_HOST=example.com,www.example.com`), the virtual host configuration file must exist for each hostname. If you would like to use the same configuration for multiple virtual host names, you can use a symlink:

```console
{ echo 'server_tokens off;'; echo 'client_max_body_size 100m;'; } > /path/to/vhost.d/www.example.com
ln -s /path/to/vhost.d/www.example.com /path/to/vhost.d/example.com
```

#### Per-VIRTUAL_HOST default configuration

If you want most of your virtual hosts to use a default single configuration and then override on a few specific ones, add those settings to the `/etc/nginx/vhost.d/default` file. This file will be used on any virtual host which does not have a `/etc/nginx/vhost.d/{VIRTUAL_HOST}` file associated with it.

#### Per-VIRTUAL_HOST location configuration

To add settings to the "location" block on a per-`VIRTUAL_HOST` basis, add your configuration file under `/etc/nginx/vhost.d` just like the previous section except with the suffix `_location`.

For example, if you have a virtual host named `app.example.com` and you have configured a proxy_cache `my-cache` in another custom file, you could tell it to use a proxy cache as follows:

```console
docker run -d -p 80:80 -p 443:443 -v /path/to/vhost.d:/etc/nginx/vhost.d:ro -v /var/run/docker.sock:/tmp/docker.sock:ro nginxproxy/nginx-proxy
{ echo 'proxy_cache my-cache;'; echo 'proxy_cache_valid  200 302  60m;'; echo 'proxy_cache_valid  404 1m;' } > /path/to/vhost.d/app.example.com_location
```

If you are using multiple hostnames for a single container (e.g. `VIRTUAL_HOST=example.com,www.example.com`), the virtual host configuration file must exist for each hostname. If you would like to use the same configuration for multiple virtual host names, you can use a symlink:

```console
{ echo 'proxy_cache my-cache;'; echo 'proxy_cache_valid  200 302  60m;'; echo 'proxy_cache_valid  404 1m;' } > /path/to/vhost.d/app.example.com_location
ln -s /path/to/vhost.d/www.example.com /path/to/vhost.d/example.com
```

#### Per-VIRTUAL_HOST location default configuration

If you want most of your virtual hosts to use a default single `location` block configuration and then override on a few specific ones, add those settings to the `/etc/nginx/vhost.d/default_location` file. This file will be used on any virtual host which does not have a `/etc/nginx/vhost.d/{VIRTUAL_HOST}_location` file associated with it.

#### Per-VIRTUAL_HOST `server_tokens` configuration
Per virtual-host `servers_tokens` directive can be configured by passing appropriate value to the `SERVER_TOKENS` environment variable. Please see the [nginx http_core module configuration](https://nginx.org/en/docs/http/ngx_http_core_module.html#server_tokens) for more details.

### Troubleshooting

In case you can't access your VIRTUAL_HOST, set `DEBUG=true` in the client container's environment and have a look at the generated nginx configuration file `/etc/nginx/conf.d/default`:

```console
docker exec <nginx-proxy-instance> cat /etc/nginx/conf.d/default
```
Especially at `upstream` definition blocks which should look like:

```Nginx
# foo.example.com
upstream foo.example.com {
	## Can be connected with "my_network" network
	# Exposed ports: [{   <exposed_port1>  tcp } {   <exposed_port2>  tcp } ...]
	# Default virtual port: <exposed_port|80>
	# VIRTUAL_PORT: <VIRTUAL_PORT>
	# foo
	server 172.18.0.9:<Port>;
	# Fallback entry
	server 127.0.0.1 down;
}
```

The effective `Port` is retrieved by order of precedence:
1. From the `VIRTUAL_PORT` environment variable
1. From the container's exposed port if there is only one
1. From the default port 80 when none of the above methods apply

### Contributing

Before submitting pull requests or issues, please check github to make sure an existing issue or pull request is not already open.

#### Running Tests Locally

To run tests, you just need to run the command below:

```console
make test
```

This commands run tests on two variants of the nginx-proxy docker image: Debian and Alpine.

You can run the tests for each of these images with their respective commands:

```console
make test-debian
make test-alpine
```

You can learn more about how the test suite works and how to write new tests in the [test/README.md](test/README.md) file.
