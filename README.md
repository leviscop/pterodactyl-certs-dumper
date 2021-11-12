# traefik-certs-dumper

Dumps Let's Encrypt certificates of a specified domain to `.pem` and `.key` files which Traefik stores in `acme.json`.

### Basic setup
Mount your ACME folder into `/traefik` and output folder to `/output`. Here's an example for docker-compose:
```yaml
version: '3.7'

services:
  certdumper:
    image: humenius/traefik-certs-dumper:latest
    volumes:
    - ./traefik/acme:/traefik:ro
    - ./output:/output:rw
    environment:
    - DOMAIN=example.org
```

### Dump all certificates
The environment variable `DOMAIN` can be left out if you want to dump all available certificates.
```yaml
version: '3.7'

services:
  certdumper:
    image: humenius/traefik-certs-dumper:latest
    volumes:
    - ./traefik/acme:/traefik:ro
    - ./output:/output:rw
    # Don't set DOMAIN
    # environment:
    # - DOMAIN=example.org
```

### Automatic container restart
If you want to have containers restarted after dumping certificates into your output folder, you can specify their names as comma-separated value and pass them through via optional parameter `-r | --restart-containers`. In this case, you must pass the Docker socket (or override `$DOCKER_HOST` if you use a Docker socket proxy). For instance:
```yaml
version: '3.7'

services:
  certdumper:
    image: humenius/traefik-certs-dumper:latest
    command: --restart-containers container1,container2,container3
    volumes:
    - ./traefik/acme:/traefik:ro
    - ./output:/output:rw
    - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
    - DOMAIN=example.org
```

It is also possible to restart Docker services. You can specify their names exactly like the containers via the optional parameter `--restart-services`. The services are updated with the command `docker service update --force <service_name>` which restarts all tasks in the service.

### Change ownership of certificate and key files
If you want to change the onwership of the certificate and key files because your container runs on different permissions than `root`, you can specify the UID and GID as an environment variable. These environment variables are `OVERRIDE_UID` and `OVERRIDE_GID`. These can only be integers and must both be set for the override to work. For instance:
```yaml
version: '3.7'

services:
  certdumper:
    image: humenius/traefik-certs-dumper:latest
    command: --restart-containers container1,container2,container3
    volumes:
    - ./traefik/acme:/traefik:ro
    - ./output:/output:rw
    - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
    - DOMAIN=example.org
    - OVERRIDE_UID=1000
    - OVERRIDE_GID=1000
```

### Extract multiple domains
This Docker image is able to extract multiple domains as well.
Use environment variable `DOMAIN` and add you domains as a comma-separated list.
After certificate dumping, the certificates can be found in the domains' subdirectories respectively.
(`/output/DOMAIN[i]/...`)
If you specify a single domain, the output folder remains the same as in previous versions (< v1.3 - `/output`).
```yaml
version: '3.7'

services:
  certdumper:
    image: humenius/traefik-certs-dumper:latest
    volumes:
    - ./traefik/acme:/traefik:ro
    - ./output:/output:rw
    - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      DOMAIN: example.com,example.org,example.net,hello.example.in
```

### Health Check
This Docker image does reports its health status.
The process which monitors `run.sh` reports back `1` when it malfunctions and `0` when it is running inside Docker container.
Normally, it's embedded in the Dockerfile which means without further ado, this works out of the box. However, if you want to specify more than one health check, you can set them via `docker-compose`.
```yaml
version: '3.7'

services:
  certdumper:
    image: humenius/traefik-certs-dumper:latest
    volumes:
    - ./traefik/acme:/traefik:ro
    - ./output:/output:rw
    - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      DOMAIN: example.com,example.org,example.net,sub.domain.ext
    healthcheck:
      test: ["CMD", "/usr/bin/healthcheck"]
      interval: 30s
      timeout: 10s
      retries: 5
```

### Merging private key and public certificate in one .pem
Load balancers like [HAProxy](http://www.haproxy.org/) need both private key and public certificate to be concatenated to one file. In this case, you can set the environment variable `COMBINED_PEM` to a desired file name ending with file extension `*.pem`. Each time `traefik-certs-dumper` dumps the certificates for specified `DOMAIN`, this script will create a `*.pem` file named after `COMBINED_PEM` in each domain's folder respectively.
```yaml
version: '3.7'

services:
  certdumper:
    image: humenius/traefik-certs-dumper:latest
    volumes:
    - ./traefik/acme:/traefik:ro
    - ./output:/output:rw
    - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      DOMAIN: example.com,example.org,example.net,hello.example.in
      COMBINED_PEM: my_concatted_file.pem
```

