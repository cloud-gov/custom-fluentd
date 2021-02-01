# Custom fluentd

A customized Docker image based on [Fluentd's image](https://hub.docker.com/r/fluent/fluentd/).

## Usage

Modify the `fluent.conf` file in this directory as needed, and build this image.

```bash
~$ docker build -t {user-name}/custom-fluentd -f Dockerfile .
```

Push the image to Dockerhub:

```bash
~$ docker push {user-name}/custom-fluentd:tagname
```

Reference the image in your `manifest.yml` or `vars.yml` file when deploying an app using this image.

