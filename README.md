# Template for running ROS1 projects in docker containers

Build container

```bash
cd ros1-docker
docker compose --file docker-compose.yaml --env-file .env build
```

Run container

```bash
# In one terminal,
cd ros1-docker && docker compose up

# In subsequent terminals,
docker exec -it ros1-docker-dev-1 bash
```
