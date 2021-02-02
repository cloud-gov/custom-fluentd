# Custom fluentd application & set up

This repo and instructions will guide you through the set up of a instance of [fluentd](https://www.fluentd.org/) for capturing logs from clpud.gov apps and [saving them to S3](https://docs.fluentd.org/how-to-guides/apache-to-s3). You will need to build and deploy a cusomized version of the fluentd Docker image. Instructions on how to do this are in the [`/docker`](docker) directory.

Note - this repo is still a work in progress.

## Create a new S3 service

```bash
~$ cf create-service s3 basic {service-name}
```
## Deploy the fluentd log drain app

In the [`/fluentd`](fluentd) directory, copy `vars.yml.example` to `vars.yml` and fill in the service name, and an internal route for the app. The fluentd log drain app must run on an `*.apps.internal` domain. Push the fluentd app without starting it to bind app to S3 service and generate credentials. (Note - you may need to explicitly bind the app to S3 service to generate credentials for vars.yml file).

```bash
~$ cf push --no-start --vars-file vars.yml
~$ cf bind-service log-drain-endpoint {service-name} # if needed, to generate service credentials
~$ cf env log-drain-endpoint # to get service credentials for vars.yml file
```

Set the remaining environmental variables for fluentd app in `vars.yml` file. Re-push fluentd app to pick up new environmental variables for the S3 service.

```bash
~$ cf push --vars-file vars.yml
```
## Deploy the log drain proxy

In the [`/proxy`](proxy) directory, copy `vars.yml.example` to `vars.yml` and fill in values for the environmental variables and route for the proxy app. The proxy should have a public route on the `*.app.cloud.gov` domain. The `proxied_route` value is the internal route for the fluentd app you set up in the prior set. The `proxied_port` is always 8080.

The proxy app uses HTTP basic authentication to restrict access to the backend fluentd log drain. To ensure this works properly, copy the `.htpasswd.example` file to `.htpasswd` and fill in the placeholders with a user name and hashed password. You'll need to hash the password generator or tool like [htpasswd](https://httpd.apache.org/docs/2.4/programs/htpasswd.html). Once you have updated the `.htpasswd` file, you ar eready to push the app.


```bash
~$ cf push --vars-file vars.yml
```

## Add network policy

Add a new network policy to allow proxy app to talk to the fluentd app. A network policy is required to allow container-to-container networking for apps running on an internal route. Note - the default port and protocol are `tcp` and `8080`.

```bash
~$ cf add-network-policy log-drain-proxy log-drain-endpoint
```

## Set up log drain

Set up log drain service using the route for the log-drain-proxy app. Make sure to use the `https://` scheme and the port number (443) in the URL for the log drain. You'll also need to add the user name and password you created when you deployed the proxy application, and include that in the URL for the log drain target:

```bash
~$ cf cups log-drain -l https://user:password@log-drain-proxy.app.cloud.gov:443/
```

## Bind log drain service

The new log drain service needs to be bound to the app you want to capture logs for. Do this with the following commands:

```bash
~$ cf bind-service {your-app-name} log-drain
~$ cf restage {your-app-name} # to ensure any needed env variables are picked up by the app
```

## Caveats

* The current fluentd config stores files locally in a cloud.gov app instance and saves them to S3 every hour (via the `flush_interval` setting in the fluentd.conf file). Because disk space is ephemeral in cloud.gov, it is possible that logs could be lost when an app instance restarts. Use of the `flush_at_shutdown` config option mitigates this risk, but probably doesn't eliminate in completely.