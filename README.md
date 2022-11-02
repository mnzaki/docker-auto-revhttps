# Notes

- This is a docker nginx config based on https://github.com/matthiasnoback/nginx-multi-host-docker which itself seems to be based on https://github.com/jwilder/nginx-proxy

- Our copy has various modifications:
    - `nginx-data` service is simplified to allow for more streamlined updates
      to the template (nginx.tmpl) by bind mounting `./nginx-data`
    - `nginx-static` and `nginx-certs` volumes to hold static files and certs
      respectively
    - `nginx-vhost` dir in this repo contains per virtual host configs
    - `nginx.tmpl` handles new "protocol" "http+static", and adds advanced logging
    - `nginx` service logs directly to graylog (see `infra/docker-graylog`)

# Setup for running multiple websites on a single server

- Uses nginx as a proxy server, and the [docker-gen](https://github.com/jwilder/docker-gen) and [docker-letsencrypt-nginx-proxy-companion](https://github.com/jwilder/docker-letsencrypt-nginx-proxy-companion) container utilities.
- Serves secure websites with automatically generated and renewed [Let's Encrypt](http://letsencrypt.org/) certificates.
- Only exposes one web server, connects to other web servers using a custom network.

## Usage

Do this once on your server:
```sh
docker network create --driver bridge nginx-proxy
```

1. Now *for every service* you want to expose on the same server, edit it's
definition in its own `docker-compose.yml` file and add the following:

```
services:
    web:
      environment:
        - VIRTUAL_HOST=your.hostname.com
        - VIRTUAL_NETWORK=nginx-proxy
        - LETSENCRYPT_HOST=your.hostname.com
        - LETSENCRYPT_EMAIL=your@email.com
      networks:
        - proxy-tier

networks:
    proxy-tier:
        external:
            name: nginx-proxy
```

2. Deploy the service by running `docker-compose`
```
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

