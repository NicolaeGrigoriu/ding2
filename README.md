# Ding2
Ding2 is a continuation of [ding.TING](http://ting.dk/content/om-dingting)
[Drupal](http://drupal.org/project/drupal) distribution for libraries as part
of the [TING concept](http://ting.dk).

# Installation
This README assumes that you have install a configured your server with a
working Apache/Nginx, APC, Memcached, PHP 5.4 and Varnish 3.x (optional). The
stack should be optimized to run a Drupal site.

## Dependencies
* [Drupal 7.23](https://drupal.org/drupal-7.23-release-notes) - latest stable
  version of Drupal Core that ding2 have been tested on and the last stable
  release when this was written.
* [Drush 6.1.0](https://github.com/drush-ops/drush) - latest release when this
  was written. See its README for installation instructions.

# Build instructions
The reset of this document explains how to download Drupal and patch the core
to run a Ding2 based site.

## Drush utils (this is a most)
Ding2uses nested makefiles (each module have its own dependencies), which
results in projects and libraries being download more than once with the
default drush installation. You can work around this by cloning the
[drush-ding2-utils](http://github.com/ding2tal/drush-ding2-utils) into your
.drush folder.
```sh
  ~$ cd ~/.drush
  ~$ git clone http://github.com/ding2tal/drush-ding2-utils.git drush-ding2-utils
```

## Drupal
Go into your web-root (from now on named DRUPAL) and execute this drush command
to download a fresh copy of Drupal version 7.23. IF you omit the version number
the newest version of Drupal will be downloaded.
```sh
  ~$ drush dl drupal-7.23
  ~$ mv drupal-7.23/* .
  ~$ mv drupal-7.23/.* .
  ~$ rm -r drupal-7.23
```

### Patches
You need to apply a set of patches to Drupal core to run the installation and
the site after the installation. To apply the patch go into your Drupal
root path and execute the commands below.

This [patch](https://drupal.org/node/1232346) fixes a problem with recursive menu rebuilds.
```sh
  ~$ wget -qO- http://drupal.org/files/menu-get-item-rebuild-1232346-22_0.patch | patch -p1
```

This [patch](https://drupal.org/node/1879970) ensure that communication with
web-services that runs OpenSSL v1.0.x or newer works.
```sh
  ~$ wget -qO- http://drupal.org/files/ssl-socket-transports-1879970-13.patch | patch -p1
```

__Optional__,but recommended patch that ensures that Ajax errors only are
displayed when not in readystate 4. So when the user presses enter to perform a
search before auto-complete Ajax is call is completed an error will not be
displayed.
```sh
  ~$ wget -qO- https://drupal.org/files/issues/autocomplete-1232416-17-7x.patch | patch -p1
```

## Build Ding2 installation profile
Ding2 comes in the form of a Drupal installation profile and the first step is
to build that profile. So go into your Drupal installations _profiles_ folder.
```sh
  ~$ cd DRUPAL/profiles
```

Clone the ding2 repository from http://github.com/ding2tal/ding2.
```sh
  ~$ git clone git@github.com:ding2tal/ding2.git ding2
```

### Production
Now that you have cloned the installation profile you need to run the build
process using drush make. It will download all the modules and the theme from
the different repositories at http://github.com/ding2tal
```sh
  ~$ cd DRUPAL/profiles/ding2
  ~$ drush --ding2-only-once --strict=0 make --concurrency=1 --no-core --contrib-destination=. ding2.make
```

### Development
If you want a developer version with _working copies_ of the Git repositories,
run this command instead. It is because drush automatically deletes _.git_
folders after it has cloned the repositories and by adding _--working-copy_, it
will not delete these.
```sh
  ~$ drush --ding2-only-once --strict=0 make --concurrency=1 --no-core --working-copy --contrib-destination=. ding2.make
```

Next goto your sites URL and run the ding2 installation profile and fill out
all the questions.

### Note
The fix in [drush-ding2-utils](http://github.com/ding2tal/drush-ding2-utils)
uses drush cache and to build the site more than once within 10 min of each
other you will need to clear the cache. This also applies if the build fails
and you need to rebuild.
```sh
  ~$ drush cc drush
```

## Alternative installation method
If you are using an deployment system you may not want to patch Drupal core
manually in a production environment.
```sh
  ~$ wget https://github.com/ding2tal/ding2/blob/development/drupal.make
  ~$ drush --ding2-only-once --strict=0 make --concurrency=1 --working-copy --contrib-destination=profiles/ding2/ drupal.make htdocs
```

# Post installation
After you have finished the installation there are some performance optimization
that you should put into your settings.php file.

## Drupal cache
This ensures that caching is enforced and that it can not be disabled in the
administration interface.

```php
  $conf['cache'] = 1;
  $conf['block_cache'] = 1;
  $conf['preprocess_css'] = 1;
  $conf['preprocess_js'] = 1;

  // Ensures that form data is not moved out of the database. It's important to
  // keep this in non-volatile memory (e.g. the database).
  $conf['cache_class_cache_form'] = 'DrupalDatabaseCache';
```

## Varnish
This project assumes that you are using Varnish as a revers proxy and the
project comes with a specially design VCL file, so that authenticated library
users can be served cached pages (ding_varnish). It also allows Varnish to have
paths purged when content is edited (varnish and expire modules).

The other varnish configurations (not listed here) are added by ding_base
feature with the strong arm module.
```php
  // Tell Drupal it's behind a proxy.
  $conf['reverse_proxy'] = TRUE;

  // Tell Drupal what addresses the proxy server(s) use.
  $conf['reverse_proxy_addresses'] = array('127.0.0.1');

  // Bypass Drupal bootstrap for anonymous users so that Drupal sets max-age < 0.
  $conf['page_cache_invoke_hooks'] = FALSE;

  // Set varnish configuration.
  $conf['varnish_control_key'] = 'THE KEY';
  $conf['varnish_socket_timeout'] = 500;

  // Set varnish server IP's sperated by spaces
  $conf['varnish-control-terminal'] = 'IP:6082 IP:6082';
```

If you do not use varnish, you should disable varnish, exipre and ding_varnish
modules as they may give you problems.

## APC
This optimization assumes that you have APC installed on your server.

__More information on the way__


## Memcache
This optimization assumes that you have memcached installed on your server.
Alternatively you can use redis as a key/value store, but it will not give you
advantages as the more advanced stuff that redis as is not used by Drupal. So
from a performance point it's more what you are use to setup.

```php
  $conf += array(
    'memcache_extension' => 'Memcache',
    'show_memcache_statistics' => 0,
    'memcache_persistent' => TRUE,
    'memcache_stampede_protection' => TRUE,
    'memcache_stampede_semaphore' => 15,
    'memcache_stampede_wait_time' => 5,
    'memcache_stampede_wait_limit' => 3,
    'memcache_key_prefix' => YOUR_SITE_NAME,
  );

  $conf['cache_backends'][] = 'profiles/ding2/modules/contrib/memcache/memcache.inc';
  $conf['cache_default_class'] = 'MemCacheDrupal';

  // Configure cache servers.
  $conf['memcache_servers'] = array(
    '127.0.0.1:11211' => 'default',
  );
  $conf['memcache_bins'] = array(
    'cache' => 'default',
  );
```
