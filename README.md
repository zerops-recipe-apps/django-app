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
- Since the Django app will run behind our HTTP balancer proxy, add your domain/subdomains to `recipe/settings.py` `CSRF_TRUSTED_ORIGINS` setting or add `APP_DOMAIN` secret variable (in recipe YAML, `app` service section)
- Disable public access to Adminer or remove it altogether (remove service `adminer` from recipe YAML)

<br/>
<br/>


## Integration Guide

<!-- #ZEROPS_EXTRACT_START:integration-guide# -->

If you want to modify your existing Django app to efficiently run on Zerops, these are the general steps we took:

### 1. Add `zerops.yaml`
Add [zerops.yaml](https://github.com/zerops-recipe-apps/django-app/blob/main/zerops.yaml) to your repository, our example includes idempotent migrations, static files collection and optimized build process.

### 2. Add S3 Storage
Run `pip install django-storages` and change storage settings section in your `project/settings.py` to support S3 compatible Object Storage file system (more info [here](https://django-storages.readthedocs.io/en/latest/backends/amazon-S3.html))

### 3. Utilize Zerops Environment Variables
Utilize Zerops [environment variables](https://github.com/zerops-recipe-apps/django-app/blob/main/zerops.yaml#L18-L32) and [secrets](https://github.com/zerops-recipe-apps/django-app/blob/main/zerops-project-import.yaml#L12-L15) to set up S3 for file system, database access, mailer and trusted hosts to work with reverse proxy load balancer

### 4. Use Init Commands to Pre-heat the Runtime Containers
Add init commands for your deployments to [migrate database and collect static images](https://github.com/zerops-recipe-apps/django-app/blob/main/zerops.yaml#L34-L39)


<!-- #ZEROPS_EXTRACT_END:integration-guide# -->

## Understand Zerops Core Concepts
If you want to try integrating Zerops from scratch on a new Laravel project, check our [step-by-step tutorial](https://docs.zerops.io/frameworks/laravel/introduction) which demonstrates how to use Zerops effectively with Laravel.

<br/>

## Tips and Others

<!-- #ZEROPS_EXTRACT_START:knowledge-base# -->

_TODO_

<!-- #ZEROPS_EXTRACT_END:knowledge-base# -->

<br/>
<br/>

Need help setting your project up? Join [Zerops Discord community](https://discord.com/invite/WDvCZ54).
