# Docker Swarm Stack with Traefik solution

## Architect diagram

![architect diagram](imgs/swarm_stack.png)

## Traefik service act as load-balancer

There are 2 Traefik instances to handle both fronttier and backtier services

**Note:**
- `"--constraints=tag==traefik-fronttier"` controls only the containers with `traefik-tags=traefik-fronttier` tags will be controlled by the `fronttier` controller
- `"--constraints=tag==traefik-backtier"` controls only the containers with `traefik-tags=traefik-backtier` tags will be controlled by the `backtier` controller
- `restart_policy` set to `condition: any` will all these 2 `traefik` container/service to auto restart after node reboot or docker daemon reboot.
- `constraints: [node.role == manager]` will force these 2 `traefik` services to run only on Swarm Manager role nodes


```yml
version: '3.3'

networks:
   frontend:
       driver: overlay
       attachable: true

volumes:
    data:

services:
  fronttier:
    image: devmr1oktodock1:5000/traefik:1.7
    command:
      - "--docker"
      - "--docker.swarmmode=true"
      - "--docker.domain=docker.localhost"
      - "--docker.watch=true"
      - "--docker.exposedbydefault=true"
      - "--docker.endpoint=unix:///var/run/docker.sock"
      - "--constraints=tag==traefik-fronttier"
      - "--web"
    ports:
      - "80:80"      # The HTTP port
      - "443:443"
      - "8000:8080" # API
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock  # So that Traefik can listen to the Docker events
    networks:
      - frontend
    labels:
      - "traefik.enable=false"
    deploy:
      placement:
        constraints: [node.role == manager]
      restart_policy:
        #condition: on-failure
        condition: any

  backtier:
    image: devmr1oktodock1:5000/traefik:1.7
    command:
      - "--docker"
      - "--docker.swarmmode=true"
      - "--docker.domain=docker.localhost"
      - "--docker.watch=true"
      - "--docker.exposedbydefault=true"
      - "--docker.endpoint=unix:///var/run/docker.sock"
      - "--constraints=tag==traefik-backtier"
      - "--web"
      - "--loglevel=DEBUG"
    ports:
      - "7180:80"      # The HTTP port
      - "7443:443"
      - "7880:8080" # API
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock  # So that Traefik can listen to the Docker events
    networks:
      - frontend
    labels:
      - "traefik.enable=false"
    deploy:
      placement:
        constraints: [node.role == manager]
      restart_policy:
        condition: any

```