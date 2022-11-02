# Automatic Reverse Proxy for multiple docker-compose installations

- Generates nginx configurations based on `nginx-data/nginx.tmpl`
- Generates and automatiically renews [Let's Encrypt](http://letsencrypt.org/) certificates.
- `nginx-static` and `nginx-certs` volumes hold static files and certs respectively
- `nginx-vhost` dir may contain per-virtual-host configs
- `nginx.tmpl` handles new "protocol" "http+static", and adds advanced logging
- `nginx` service logs directly to graylog (see `infra/docker-graylog`)

# Setup for running multiple websites on a single server

## Usage

First create an external volume and network that all the projects (different
`docker-compose.yml` files) will share with `docker-auto-revhttps`.
This needs to be done only once on the physical server:

```sh
docker network create --driver bridge nginx-proxy
docker volume create nginx-static
```

Now, for every project, edit its `docker-compose.yml` file and declare the external
network and volume by adding this at the top level:

```yml
networks:
  nginx-proxy:
    external: true
volumes:
  nginx-static
    external: true
```

Now suppose this docker-compose.yml file has a few different services and one of
them is called `web` which listens on its port 80 as the entry point for the
project. We want this service to be picked up by `auto-revhttps` so we add the
following environment variables and networks configuration to it:

> Note the `[...]` implies snipped out details

```yml
services:
  web:
    networks:
      - default
      - nginx-proxy
    environment:
      [...]
      - VIRTUAL_HOST=your.hostname.com
      - VIRTUAL_PORT=80
      - VIRTUAL_NETWORK=nginx-proxy
    [...]
  [...]
[...]

```

Note that we add the `default` network explicitly as otherwise the `web` service
will not be able to access the other services within the project.

If the `web` service has a `ports` definition and uses port `80` or `443` of the
host then you *must* remove that definition otherwise it will conflict with
`auto-revhttps` itself.

Furthermore, to generate SSL certificates, then add the following environment
variables as well:
```yml
      - LETSENCRYPT_HOST=your.hostname.com(,another.hostname.com,...)
      - LETSENCRYPT_EMAIL=your@email.com
```

`LETSENCRYPT_HOST` can either be 1 hostname, or several comma separated
hostnames.

And finally deploy `your-project`:
```sh
cd your-project
docker-compose up -d --force-recreate --no-build
```


It may take a minute or so before the secure site can be reached (because the certificate has to be created first).

# Tips and Tricks
Some things can be improved, until then please note:

## Changing the nginx config template
If you change `nginx-data/nginx.tmpl` then push, you need to trigger a rebuild
of the config files (which will also reload nginx as a side effect):
```
$ docker-compose restart nginx-gen
```

## Changing the nginx config for one virtual-host
If you update the configs in `nginx-vhost` then push, you need to ask nginx
to reload the config:
```
$ docker-compose exec nginx "nginx -r && nginx -s reload"
```

## Forcing letsencrypt certbot renewal
If you need to force renewal of the letsencrypt certificates:
```
docker-compose exec nginx-letsencrypt /app/force_renew
```

# History
- This is a docker nginx config based on https://github.com/matthiasnoback/nginx-multi-host-docker which itself seems to be based on https://github.com/jwilder/nginx-proxy
- Uses nginx as a proxy server, and the [docker-gen](https://github.com/jwilder/docker-gen) and [docker-letsencrypt-nginx-proxy-companion](https://github.com/jwilder/docker-letsencrypt-nginx-proxy-companion) container utilities.

