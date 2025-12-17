# Zerops x Django

[Django](https://www.djangoproject.com/) is a high-level Python web framework that encourages rapid development and clean, pragmatic design. This recipe aims to showcase few advanced Django concepts and how to integrate them with [Zerops](https://zerops.io), all through a simple file upload demo application.

![django](https://github.com/zeropsio/recipe-shared-assets/blob/main/covers/svg/cover-django.svg)

<br />

## Deploy on Zerops
You can either click the deploy button to deploy directly on Zerops, or manually copy the [import yaml](https://github.com/zerops-recipe-apps/django-app/blob/main/zerops-project-import.yaml) to the import dialog in the Zerops app.

[![Deploy on Zerops](https://github.com/zeropsio/recipe-shared-assets/blob/main/deploy-button/green/deploy-button.svg)](https://app.zerops.io/recipe/django)

<br/>

## Recipe features

- **Load balanced** Django web app running on **Zerops Python** service
- Served by production-ready application server **[Gunicorn](https://gunicorn.org/)**
- Zerops **PostgreSQL 16** service as database
- Zerops **Object Storage** (S3 compatible) service as file system
- Automatic Django **database migrations**, **static files collection** and **superuser seeding**
- Utilization of Zerops built-in **environment variables** system
- Logs accessible through Zerops GUI
- **[Mailpit](https://github.com/axllent/mailpit)** as **SMTP mock server**
- **[Adminer](https://www.adminer.org)** for **quick database management** tool
- Unlocked development experience:
  - Access to database and mail mock through Zerops project VPN (`zcli vpn up`)
  - Prepared `.env.dist` file (`cp .env.dist .env` and change ***** secrets found in Zerops GUI)

<br/>

## Production vs. development

Base of the recipe is ready for production, the difference comes down to:

- Use highly available version of the PostgreSQL database (change `mode` from `NON_HA` to `HA` in recipe YAML, `db` service section)
- Use at least two containers for Django service to achieve high reliability and resilience (add `minContainers: 2` in recipe YAML, `app` service section)
- Use production-ready third-party SMTP server instead of Mailpit (change `MAIL_` secret variables in recipe YAML `app` service)
- Since the Django app will run behind our HTTP balancer proxy, add your domain/subdomains to `recipe/settings.py` `CSRF_TRUSTED_ORIGINS` setting or change the `APP_DOMAIN_URLS` environment variable (in recipe YAML, `app` service section)
- Disable public access to Adminer or remove it altogether (remove service `adminer` from recipe YAML)

<br/>
<br/>


## Integration Guide

<!-- #ZEROPS_EXTRACT_START:integration-guide# -->

If you want to modify your existing Django app to efficiently run on Zerops, these are the general steps we took:

### 1. Add `zerops.yaml`

Add the following zerops.yaml file to the root of your repository. The zerops.yaml file defines how Zerops should build, deploy and run your application. Feel free to customize it to better suit your application's needs.

This example includes idempotent migrations, configuration via environment variables and support for remote or agent development.

```yaml
zerops:
  # Prepare `base` setup to reduce definition duplication.
  - setup: base
    run:
      # The Django app is made configurable via environment variables,
      # as this is the recommended way to deploy to Zerops.
      # Environment variables are used in `settings.py` via `os.getenv("KEY")`.
      envVariables:
        # Don't hold the logs in a buffer, write them immediately
        # as they occur.
        PYTHONUNBUFFERED: 1
        
        # Change this in production environment,
        # to match your own domain URL, e.g. `https://domain.example`.
        # Accepts comma seperated URL list, which is used to configure
        # `ALLOWED_HOSTS` and `CSRF_TRUSTED_ORIGINS` in `settings.py`. 
        APP_DOMAIN_URLS: $zeropsSubdomain

        # Database connection setup.
        DB_HOST: $db_hostname
        DB_PORT: $db_port
        DB_USER: $db_user
        DB_PASSWORD: $db_password

        # Use S3-compatible object storage as file and media storage.
        USE_S3: 1
        S3_ACCESS_KEY_ID: $storage_accessKeyId
        S3_SECRET_ACCESS_KEY: $storage_secretAccessKey
        S3_BUCKET_NAME: $storage_bucketName
        S3_ENDPOINT_URL: $storage_apiUrl

        # Send emails to the Mailpit by default,
        # feel free to change or override this
        # via the `_OVERRIDE` environment variables.
        MAIL_HOST: ${MAIL_HOST_OVERRIDE:-mailpit}
        MAIL_PORT: ${MAIL_PORT_OVERRIDE:-1025}
      # Make the service available on port 8000
      # from the outside of the container.
      ports:
        - port: 8000
          httpSupport: true

  # Define `prod` setup that should be used
  # for running app of stage and production deployments.
  - setup: prod
    extends: base
    build:
      # Use lightweight Alpine Linux as base for app deployment. 
      os: alpine
      base: python@3.12
      # Choosing to install the dependencies system-wide without
      # Python virtual environment. That's why it's necessary to include
      # the `requirements.txt` for the `prepareCommands` phase,
      # where the runtime container image is created (runtime dependencies installation, ...).
      # 
      # Alternative solution would be to install
      # all libraries and other dependencies to `.venv` directory
      # and deploying the app with it. Similar to how other
      # interpreted languages are deployed:
      #   - PHP with Composer and `vendor`
      #   - Node.js with `npm` and `node_modules`
      addToRunPrepare:
        - requirements.txt
      deployFiles:
        - files/
        - recipe/
        # Include the `manage.py` for maintenance reasons. 
        - manage.py
        - gunicorn.conf.py
    deploy:
      # Check that our app is able to start during deployment
      # of a new version. This can prevent bad releases from
      # rolling out into production.
      readinessCheck:
        httpGet:
          port: 8000
          path: /health
    run:
      os: alpine
      base: python@3.12
      # Install required `tzdata` system package and all
      # Python packages system-wide.
      prepareCommands:
        - sudo apk add --no-cache tzdata
        - pip install --no-cache-dir -r requirements.txt
      initCommands:
        # Migrate the database and collect static files, every time new code is released.
        # Execute these command exactly once across all containers during upgrade,
        # more about the `zsc` utility: https://docs.zerops.io/references/zsc#execonce
        # 
        # Utilize the `$ZEROPS_appVersionId` environment variable,
        # which is different for every new app version (release of the app code).
        - zsc execOnce migrate-${ZEROPS_appVersionId} -- python manage.py migrate
        - zsc execOnce collectstatic-${ZEROPS_appVersionId} -- python manage.py collectstatic --no-input --verbosity 3
        # Create superuser for admin page.
        # Using static exec name `createsuperuser` causing this command
        # to be executed exactly once of the service existence.
        - zsc execOnce createsuperuser -- python manage.py createsuperuser --no-input --username admin --email admin@example.com || true
      # Serve the app via Gunicorn Python HTTP server: https://gunicorn.org/
      start: gunicorn recipe.wsgi

  # Define `dev` setup for remote development
  # or agentic development setups.
  # Contains source code, pre-installed packages and environment variables
  # making it ready to develop inside the runtime container.
  - setup: dev
    extends: base
    build:
      os: ubuntu
      base: python@3.12
      addToRunPrepare:
        - requirements.txt
      # Deploy all source code files.
      deployFiles:
        - ./
    run:
      # Specify Ubuntu as runtime OS for its richer tool-set.
      os: ubuntu
      base: python@3.12
      # Turn on Django debug.
      envVariables:
        DEBUG: 1
      # Pre-install system and Python packages
      # (again system-wide, see note in `prod` setup section).
      prepareCommands:
        - sudo apt-get install tzdata
        - pip install -r requirements.txt
      # The port 8000 is inherited from `base` setup,
      # making it possible to access from outside during dev-testing cycle.
```

### 2. Add S3 Storage
Run `pip install django-storages` and change storage settings section in your `project/settings.py` to support S3 compatible Object Storage file system (more info [here](https://django-storages.readthedocs.io/en/latest/backends/amazon-S3.html)).

### 3. Utilize Zerops Environment Variables
Utilize Zerops [environment variables](https://github.com/zerops-recipe-apps/django-app/blob/main/zerops.yaml#L8-L38) to set up S3 for file system, database access, mailer and trusted hosts to work with reverse proxy load balancer.

### 4. Use Init Commands to Run Migrations
Add init commands for your deployments to migrate database and collect static images, utilizing [`zsc exec-once`](https://docs.zerops.io/references/zsc#execonce):

```shell
# Utilize the `$ZEROPS_appVersionId` environment variable,
# which is different for every new app version (release of the app code).
zsc execOnce migrate-${ZEROPS_appVersionId} -- python manage.py migrate
zsc execOnce collectstatic-${ZEROPS_appVersionId} -- python manage.py collectstatic --no-input --verbosity 3

# Using static exec name `createsuperuser` causing this command
# to be executed exactly once of the service existence.
zsc execOnce createsuperuser -- python manage.py createsuperuser --no-input --username admin --email admin@example.com || true
```

<!-- #ZEROPS_EXTRACT_END:integration-guide# -->

## Understand Zerops Core Concepts
If you want to try integrating Zerops from scratch on a new Laravel project, check our [step-by-step tutorial](https://docs.zerops.io/frameworks/laravel/introduction) which demonstrates how to use Zerops effectively with Laravel.

<br/>

## Tips and Others

<!-- #ZEROPS_EXTRACT_START:knowledge-base# -->

### Database Permissions for Testing Remotely (`manage.py test`)

Command `python manage.py test` creates fresh database for running the tests.
The [default database user](https://docs.zerops.io/postgresql/how-to/manage#default-database-and-user) doesn't have permissions to create database by default.

You can add the `CREATEDB` Postgres permissions with the following statement, after logging in as the `postgres` [super-user](https://docs.zerops.io/postgresql/how-to/manage#installing-plugins-requires-superuser):

```postgresql
ALTER USER db CREATEDB;
```


<!-- #ZEROPS_EXTRACT_END:knowledge-base# -->

<br/>
<br/>

Need help setting your project up? Join [Zerops Discord community](https://discord.com/invite/WDvCZ54).
