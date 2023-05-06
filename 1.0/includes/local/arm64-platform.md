### M1/M2 Macs
#### Platform issue
Apple's latest M1/M2 platforms should use images built for **arm64**, for example **linux/arm64**.

By default `docker compose up -d` will pull **amd64** images from Docker Hub. With these images your containers will run noticable slower, because of the platform mismatch.

Also if you visit `Images` section of your Docker Desktop, you will see warnings regarding some of the images being meant for AMD64 platform.

#### Fix the issue
While it is not possible to use **arm64** for all containers, it is possible to do so for most important ones.

Go to `docker-compose.yml` find `php` container definition. There add a new line `platform: linux/arm64`. So it looks like this:
```
  php:
    image: wodby/drupal-php:$PHP_TAG
    platform: linux/arm64
    container_name: "${PROJECT_NAME}_php"
    ...
```

Run `docker compose up -d`.

**Note:** You can do the same to `nginx` container. Unfortunatelly it is not possible with `mariadb`, `phpmyadmin`, etc. Becase those images are built only for **amd64** platform.

