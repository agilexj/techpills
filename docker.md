## Override Container Entrypoint

```
docker run -it --rm --entrypoint=bash <image_name>
```

* `-it` allows you to interactively use a shell inside the container. These are two flags combined, a single-dash (-) is used for short options (one-letter flags).
* `--rm` tells Docker to automatically remove the container after it exits. A double-dash (--) is used for long options (full-word flags).

## Docker Volumes 
Check for existing volumes:
```
docker volume ls -q
```

Remove existing volumes
```
docker volume rm $(docker volume ls -q)
```

## Inspect container
```
docker inspect <container_name> | les
```