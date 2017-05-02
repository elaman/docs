# Drupal 7 stack

StackHub URL: https://cloud.wodby.com/#/hub/35e3e058-936b-4695-9871-08c855aae898/detail

* [Overview](#overview)
* [Containers](#containers)
  * [MariaDB](#mariadb)
  * [Solr](#solr)
  * [Redis](#redis)
  * [Varnish](#varnish)
* [Mail Transfer Agent](#mail-transfer-agent)
* [Local environment](#local-environment)
* [Drupal settings](#drupal-settings)
* [SSH](#ssh)
* [Cron](#cron)
* [Drush](#drush)
* [Drupal Console](#drupal-console)
* [Multi-site](#multi-site)
* [Redirects](#redirects)
* [Deploy via composer](#deploy-via-composer)
* [Environment variables](#environment-variables)
* [Import](#import)
 
## Overview

[wodby/drupal-nginx]: https://github.com/wodby/drupal-nginx
[wodby/drupal-php]: https://github.com/wodby/drupal-php
[wodby/mariadb]: https://github.com/wodby/mariadb
[wodby/redis]: https://github.com/wodby/redis
[wodby/drupal-varnish]: https://github.com/wodby/drupal-varnish
[wodby/drupal-solr]: https://github.com/wodby/drupal-solr
[wodby/adminer]: https://github.com/wodby/adminer
[phpmyadmin/phpmyadmin]: https://hub.docker.com/r/phpmyadmin/phpmyadmin
[mailhog/mailhog]: https://hub.docker.com/r/mailhog/mailhog

| Container | Image |
| --------- | ----------- |
| Nginx | [wodby/drupal-nginx] |
| PHP | [wodby/drupal-php] |
| [MariaDB](#mariadb) | [wodby/mariadb] |
| [Redis](#redis) | [wodby/redis] |
| [Varnish](#varnish) | [wodby/drupal-varnish] |
| [Solr](#solr) | [wodby/drupal-solr] |
| Mailhog | [mailhog/mailhog] |
| PhpMyAdmin | [phpmyadmin/phpmyadmin] |
| Adminer | [wodby/adminer] |

## Containers

### MariaDB

If you want to access the database outside of the Wodby infrastructure you will have to use SSH tunnel via the main container:

1. Set up SSH tunnel on port `53306` (you can change it). You can find `<SSH Port>` on `[Instance] > Stack > SSH` page. For MySQL (port `3306` by default) use the following command:
```
$ ssh -L 53306:mariadb:3306 -p <SSH Port> wodby@<Server IP> -N
```

2. Connect to the database (mysql) via the tunnel on port `53306`:
```bash
$ mysql --protocol=TCP -P53306 -udrupal -p<MySQL password> drupal
```

### Solr

To integrate Solr with Drupal just install [Search API Solr module](https://www.drupal.org/project/search_api_solr). No additional configuration required on Solr side (it's already configured for you).

#### Connecting server

Go to `Configuration » Search and metadata » Search API` and select Service class to "Solr service". In expanded settings fieldset specify:
```
HTTP protocol: http
Solr host: <Copy Internal hostname from "[Instance] > Stack > Search engine">
Solr port: 8983
Solr path: /solr/wodby
Basic authentication:
  Username: admin
  Password: <Copy password from "[Instance] > Stack > Search engine">
```

### Redis

Install and enable [redis module](https://www.drupal.org/project/redis) to use it as an internal cache storage for Drupal.

### Varnish

1. Install and enable varnish module (use the dev version)
Enable Varnish integration from App page > Cache > Cache settings
2. Go to Home » Administration » Configuration » Development page of Drupal website and
3. Check Cache pages for anonymous users
4. Check Compress cached pages.
Check Aggregate and compress CSS files.
Check Aggregate JavaScript files.
Also, we recommend to install expire module to configure auto purge of pages when some content has been updated. After installation go to Home » Administration » Configuration » System and select External expiration at the "Module status" tab.

#### Caching rules

Varnish ignores the following GET parameters for cache id generation:

```
utm_source
utm_medium
utm_campaign
utm_content
gclid
cx
ie
cof
siteurl
```

Set header `Cache-Control:no-cache` to tell Varnish to not cache this page.

#### Headers

* `X-Varnish-Cache`: HIT or MISS, corresponds to when the cache was found or not
* `Age: 34`: age of the cache in seconds
* `X-Varnish: 65658 65623` - the first number is the ID of a request, the second is the ID of cache inside of Varnish. When operating normally the first number changes with every request of the same page and the second stays the same.

#### CLI

Grouped list with the most usual entries from different logs:
```bash
$ varnishtop
```

A histogram that shows the time taken for the requests processing:
```bash
$ varnishhist
```

Varnish stats, shows how many contents on cache hits, resource consumption, etc..:
```bash
$ varnishstat
```

Log showing requests made to the web backend server:
```bash
$ varnishlog
```

## Mail Transfer Agent

MTA server (OpenSMTPD) included to the stack by default. Additionally, you can catch all your emails by adding a mail catcher service (Mailhog).

### Guaranteed delivery of transaction emails

To make sure your transaction emails will be guaranteed delivered we recommend to use Transaction email services such as:

* <a href="http://sendgrid.com/" target="_blank">SendGrid</a> (has a free version). Read <a href="http://atendesigngroup.com/blog/send-mail-drupal-7-deliver-email-reliably-avoid-spam-folder" target="_blank">this article</a> on how to integrate SendGrid with Drupal
* <a href="https://aws.amazon.com/ses/" target="_blank">AWS SES</a>
* <a href="http://mailchimp.com/" target="_blank">Mailchimp</a>

## Local environment

For local environment we recommend using [docker4drupal](https://github.com/wodby/docker4drupal) fully consistent with this stack.

## Drupal settings

### settings.php

Wodby automatically adds include of `wodby.settings.php` to `settings.php` file inside of `sites/[SITE NAME]`. The value of `[SITE NAME]` is `default` unless you specify it distinctly on a new application deployment form. If directory doesn't exist Wodby will create it automatically.

Do not edit wodby.settings.php, all changes to this file will be reset.

The `wodby.settings.php` file contains configuration settings for integration with Wodby services such as Database, Cache storage and Reverse Caching Proxy. You can override settings specified in wodby.settings.php in your `sites/*/settings.php` file after the include of wodby.settings.php

### sites.php

Wodby automatically generates `sites.php` only when [multi-site](multi-site.md) directory specified. When the multi-site directory specified `sites.php` will contain mapping of all [domains attached via Wodby](../domains.md) to the directory.

### Files

Files for Drupal located in `/mnt/files` and symlinked to `sites/default/files`.

### Base URL

The domain marked with primary flag will be used as a `$base_url` in settings.php file and as an `-l` parameter for cron.

## SSH

The copy of PHP container runs with SSHd. You can find access information on `[Instance] > Stack > SSH`

## Cron

The copy of PHP container runs with cron and runs every hour via `drush cron -l {{BASE_URL}}`.

## Drush

PHP container comes with installed drush. You can execute drush commands remotely via drush aliases. Download drush aliases from `[Instance] > Settings > Drush` page and place them to `~/.drush`. Execute commands (replace `[tokens]` with the real values) like this:

```bash
$  drush @[organization].[application].[instance] [drush command]
```

## Drupal Console

PHP container comes with installed drupal console launcher (not the same as drupal console), the launcher used to be able run drupal console without specified the full path to it. Please note that starting Drupal Console ~1.0 you have to install it manually (via composer) per project.

## Multi-site

You can deploy your existing multi-site Drupal as separate apps (one per each site). specify Multi-site directory (e.g. if you have a directory `sites/my-drupal-site` just specify `my-drupal-site`). This directory will be used to locate <a href="settings.html#settingsphp">`settings.php`</a> file in this directory. Also, <a href="settings.html#sitesphp">`sites.php`</a> file will be created automatically with mapping of all domains attached via Wodby to this sites directory.

## Redirects

If you need to make a redirect from one domain to another you can do it by customizing configuration files of nginx or by adding the snippets below to your `settings.php` file.

### Redirect from one domain to another

```php
if (isset($_SERVER['WODBY_ENVIRONMENT_TYPE']) && $_SERVER['WODBY_ENVIRONMENT_TYPE'] == 'prod' && php_sapi_name() != "cli") {
  if ($_SERVER['HTTP_HOST'] == 'redirect-from-domain.com') {
    header('HTTP/1.0 301 Moved Permanently');
    header('Location: http://redirect-to-domain.com' . $_SERVER['REQUEST_URI']);
    exit();
  }
}
```

### Redirect from multiple domains.

```php
if (isset($_SERVER['WODBY_ENVIRONMENT_TYPE']) && $_SERVER['WODBY_ENVIRONMENT_TYPE'] == 'prod' && php_sapi_name() != "cli") {
  $redirect_from = array(
    'redirect-from-domain-1.com',
    'redirect-from-domain-2.com',
  );

  if (in_array($_SERVER['HTTP_HOST'], $redirect_from)) {
    header('HTTP/1.0 301 Moved Permanently');
    header('Location: http://redirect-to-domain.com' . $_SERVER['REQUEST_URI']);
    exit();
  }
}
```

### Redirect from HTTP to HTTPS

You can enable this redirect by checking the corresponding option on a domain edit page from the dashboard.

## Deploy via composer

Add `wodby.yml` file to your git repository root with the following content:

```yml
pipeline:
  - name: Install dependencies
    type: command
    command: composer install
    directory: $APP_ROOT
```

Read [this article](../deployment/post-deployment-scripts.md) to learn more about post-deployment scripts.

## Environment variables

| Variable  | Description |
| --------- | ----------- |
| `$APP_ROOT`               | `/var/www/html` by default |
| `$HTTP_ROOT`              | `/var/www/html` by default |
| `$WODBY_ENVIRONMENT_NAME` | |
| `$WODBY_ENVIRONMENT_TYPE` | |

## Import

### From drush archive

First, make sure you have <a href="http://www.drush.org/en/master/install/" target="_blank">Drush installed</a>, go to your Drupal website docroot and execute a command:

```bash
$ drush archive-dump
```

or

```bash
$ drush ard
```

You should see output like:

```bash
$ drush ard
Archive saved to /Users/johndoe/drush-backups/archive-dump/20150604001227/drupalapp.20150604_001228.tar.gz
/Users/johndoe/drush-backups/archive-dump/20150604001227/drupalapp.20150604_001228.tar.gz
```

Now navigate to `Apps > Deploy` and choose drush archive on the 2nd step

### From separate archives

Alternatively, you can import Drupal via separate archives for code, database and files. We support `.zip`, `.gz`, `.tar.gz`, `.tgz` and `.tar` archives.

> For big archives we recommend importing it manually after you deploy an app

### Manual

In case your Drupal website is huge it makes sense to import your database/files manually from the server. Follow these steps:

1. Deploy your Drupal website from a git repository without upload database and files
2. Once the app is deployed, go to `Stack > SSH` and copy SSH command
3. Connect the container by SSH and navigate to Drupal docroot (normally it's `/var/www/html`)
4. Copy your database archive here using `wget` or `scp`, unpack the archive
5. Import unpacked database dump using `drush sql-cli < my-db-dump.sql`
6. Now let's import your files, cd to `/mnt/files`
7. Copy your files archive here using `wget` or `scp` and unpack the archive
8. That's it! Clear app cache from the dashboard and don't forget to remove archives and extracted db dump