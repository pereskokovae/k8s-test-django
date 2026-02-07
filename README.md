# Django Site
A Dockerized Django website for experimenting with Kubernetes.

Inside the container, the Django application is run using `Nginx Unit` (not to be confused with Nginx).
The Nginx Unit server performs two roles at once
1. as a `web server`, it serves static and media files.
2. as an `application server`, it runs Python and Django.
In this setup, Nginx Unit replaces the traditional combination of Nginx + gunicorn.
You can read more about Nginx Unit in the [official documentation](https://unit.nginx.org/).

## Preparing the environment for local development

The code in this repository is fully Dockerized, so you need `Docker` to run the application. Installation instructions can be found on the official website:
- [Get Started with Docker](https://www.docker.com/get-started/)

Along with a recent version of Docker, `Docker Compose` will be installed automaticaly.
All further instructions actively use Docker Compose. 

----------

## Running the site locally

Start the detabase and the web application
```shell
docker compose up
```

In a new terminal, while the site is running, execute the following commands
```shell
docker compose run --rm web ./manage.py migrate  #create or update database tables
docker compose run --rm web ./manage.py createsuperuser  #create a superuser account
```

Done!
The site will be available at: [http://127.0.0.1:8080](http://127.0.0.1:8080). 

The django admin interface is available at: [http://127.0.0.1:8000/admin/](http://127.0.0.1:8000/admin/).

-----------

## Development workflow

All Django source files are mounted into the Docker container.
This allows Nginx Unit to immediately pick up code changes without rebuilding the Docker image — restarting Docker Compose services is enough.

### Updating the application from the main repository 
To update the application to the latest version, pull the changes from the main repository and rebuild the Docker images
```bash
git pull
docker compose build
```

After updating the code in the repository, you should also update the databases schema.
New commits may include database migrations, and without application them will not start.

To avoid guessing whether migrations are required, run the `migrate` command after each update.
```shell
docker compose run --rm web ./manage.py migrate
```
Example output:
```text
Running migrations:
  No migrations to apply.
```

### Adding a dependency

The Django image uses `pip` as the package manager with a `requirements.txt` file.
To add a new dependency, simply add it to requirements.txt and rebuild the Docker image
```bash
docker compose build web
```

Dependencies can be removed in the same way.

----------

## Environment variables

The Django image reads its configuration from environment variables
`SECRET_KEY` — a required Django secret setting. This value is used as a salt for generating hashes. It can be any string, but it must remain private. [Django documentation](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` — enables or disables Django debug mode. Accepts `TRUE` or `FALSE`. [Django documentation](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` — a Django setting that defines a list of allowed hostnames. Requests to any other host will result in a 400 error. Multiple hosts can be specified, separated by commas, for example: `127.0.0.1,192.168.0.1,site.test`. [Django documentation](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` — the connection string for the PostgreSQL database. Other databases engines are not supported. [Connection string format](https://github.com/jacobian/dj-database-url#url-schema).

----------

## Minikube (local Kubernetes)
For local Kubernetes development and testing, this project can be run in Minikube.

- Start Minikube and enable the NGINX Ingress addon.
- Apply the Kubernetes manifests from the k8s/ directory.
- Access the site via a local domain.

See the full step-by-step guide here:
[Kubernetes & Infrastructure Setup](deploy/yc-sirius-dev/README.md)


## Kubernetes Deployment
This project can be deployed to a Kubernetes cluster.

The Kubernetes setup includes:
- storing sensitive configuration in Kubernetes Secrets.
- deploying the application using manifests.
- exposing the service via Ingress in Minikube.

Detailed instructions for Kubernetes deployment and configuration can be found here:
[Kubernetes & Infrastructure Setup](deploy/yc-sirius-dev/README.md)
