---
title: Configuring Settings.php
description: Detailed information about configuring your Drupal database settings.
tags: [variables]
contributors: [mmenavas, andrewmallis]
categories: []
---
The Drupal system configuration in code is set in the `sites/default/settings.php` file.

Drupal 8 sites on Pantheon run an unmodified version of core, bundled with a custom `settings.php` file that includes the necessary `settings.pantheon.php`. If the stock `settings.php` file is used in place of the bundled file, the site will stop working on Pantheon.

For Drupal 6/7, Pantheon uses a variant of Pressflow Drupal to allow the server to automatically specify configuration settings, such as the database configuration without editing `settings.php`. Permissions are handled automatically by Pantheon, so you can customize `settings.php` like any other site code.

<div class="alert alert-danger" role="alert"><h4 class="info">Warning</h4>
<p>You should never put the database connection information for a Pantheon database within your <code>settings.php</code> file. These credentials will change. If you are having connection errors, make sure you are running Pressflow core. This is a requirement.</p>
</div>

## Pantheon Articles on settings.php

The following articles include techniques and configurations for `settings.php` on Pantheon:

- [Reading Pantheon Environment Configuration](/docs/read-environment-config) (including domain_access)
- [Installing Redis on Drupal or WordPress](/docs/redis/)
- [Platform and Custom Domains](/docs/domains/)
- [Configure Redirects](/docs/redirects/)
- [SSO and Identity Federation](/docs/sso) (LDAP TLS certificate configuration)

## Local Database Configuration for Development

Use these configuration snippets to specify a local configuration that will be ignored by Pantheon, such as database credentials.

### Drupal 8
Configure environment-specific settings within the `settings.local.php` file, which is ignored by git in our [Drupal 8 upstream](https://github.com/pantheon-systems/drops-8). Modifying the bundled `settings.php` file is not necessary, as it already includes `settings.local.php` if one exists.

    // Local development configuration.
    if (!defined('PANTHEON_ENVIRONMENT')) {
      // Database.
      $databases['default']['default'] = array(
        'database' => 'DATABASE',
        'username' => 'USERNAME',
        'password' => 'PASSWORD',
        'host' => 'localhost',
        'driver' => 'mysql',
        'port' => 3306,
        'prefix' => '',
      );
    }


The `HASH_SALT` value should also be set within `settings.local.php`. See Drush script: [Quickstart](https://github.com/pantheon-systems/drush-config-workflow/blob/master/bin/quickstart)

To use the Pantheon `HASH_SALT` in your local site (not necessary), you can get it via [Terminus](/docs/terminus):
```
terminus drush <site>.<env> -- ev 'return getenv("DRUPAL_HASH_SALT")'
```

Drupal 8 will not run locally without a hash salt, but it need not be the same one set on the Pantheon platform; any sufficiently long random string will do. Make sure to set one in `settings.local.php` :
```
$settings['hash_salt'] = '$HASH_SALT';
```

#### Trusted Host Setting
A warning within `/admin/reports/status` will appear when the `trusted_host_patterns` setting is not configured. This setting protects sites from HTTP Host header attacks. However, sites running on Pantheon are not vulnerable to this specific attack and the warning can be safely ignored. If you would like to resolve the warning, use the following configuration:
<div class="alert alert-info">
<h4 class="info">Note</h4>
<p markdown="1">Replace `^www.yoursite.com$` with custom domain(s) added within the Site Dashboard, adjusting patterns as needed. If you're using the Drupal 8 redirects from our [Configure Redirects](/docs/redirects/#redirect-to-https-and-the-primary-domain) doc, don't use this snippet as it conflicts.</p>
</div>
```
if (defined('PANTHEON_ENVIRONMENT')) {
  if (in_array($_ENV['PANTHEON_ENVIRONMENT'], array('dev', 'test', 'live'))) {
    $settings['trusted_host_patterns'][] = "{$_ENV['PANTHEON_ENVIRONMENT']}-{$_ENV['PANTHEON_SITE_NAME']}.pantheon.io";
    $settings['trusted_host_patterns'][] = "{$_ENV['PANTHEON_ENVIRONMENT']}-{$_ENV['PANTHEON_SITE_NAME']}.pantheonsite.io";


    # Replace value with custom domain(s) added in the site Dashboard
    $settings['trusted_host_patterns'][] = '^.+.yoursite.com$';
    $settings['trusted_host_patterns'][] = '^yoursite.com$';
  }
}
```


### Drupal 7

    // Local development configuration.
    if (!defined('PANTHEON_ENVIRONMENT')) {
      // Database.
      $databases['default']['default'] = array(
        'database' => 'DATABASE',
        'username' => 'USERNAME',
        'password' => 'PASSWORD',
        'host' => 'localhost',
        'driver' => 'mysql',
        'port' => 3306,
        'prefix' => '',
      );
    }

### Drupal 6

    // Local development configuration.
    if (!defined('PANTHEON_ENVIRONMENT')) {
      // Database.
      $db_url = 'mysql://username:password@localhost/databasename';
      $db_prefix = '';
    }

## Frequently Asked Questions

#### Can I delete the default.settings.php file?
Yes, but only if at least one other file (e.g. `settings.php`) is present within the `sites/default` directory. Otherwise, the existing symlink to `sites/default/files` will be invalid.

#### How can I write logic based on the Pantheon server environment?

Depending on your use case, there are three possibilities:

 - For web only actions, like redirects, check for the existence of `$_ENV['PANTHEON_ENVIRONMENT']`. If it exists, it will contain a string with the current environment (Dev, Test, or Live. See our [redirects](/docs/domains/#redirect-to-https-and-the-primary-domain) guide for examples.

    <div class="alert alert-info">
    <h4 class="info">Note</h4>
    <p markdown="1">`$_SERVER` is not generally available from the command line so [logic should check for that when used](/docs/domains/#troubleshooting), and [avoid using `$_SERVER['SERVER_NAME']` and `$_SERVER['SERVER_PORT']`](/docs/server_name-and-server_port/).</p>
    </div>

 - For actions that should take place on every environment, such as Redis caching, use the constant `PANTHEON_ENVIRONMENT`. Again, it will contain Dev, Test, or Live. See our [Redis](/docs/redis/) guide for examples.

 - For Actions that require access to protected services like Redis or the site database, you can use the `$_ENV` superglobal. Please review our guide on [Reading Pantheon Environment Configuration](/docs/read-environment-config/) for more information, or see our [Redis](/docs/redis/) guide for examples.

#### Why does Drupal report that `settings.php` is not protected? I can't change the permissions on `settings.php`.

If you do not have a `settings.php` file in your codebase, you'll see the following message on `/admin/reports/status`:

Configuration file: Not protected. The file `sites/default/settings.php` is not protected from modifications and poses a security risk. You must change the file's permissions to be non-writable.

Technically, it's possible to have a functioning Drupal site without `settings.php` on Pantheon, but this breaks compatibility with many modules and tools. Therefore, it's strongly recommended to either copy the `default.settings.php` file to `settings.php` or create an empty `settings.php` file.

#### Should I include settings.php in my site import?

It depends on your site configuration. Stripping commented-out or non-functional code from your existing `settings.php` file, leaving only known good functional configurations is a best practice and makes it easier to troubleshoot.

#### Where do I specify database credentials?

Pantheon automatically injects database credentials into the site environment; if you hard code database credentials, you will break the Pantheon workflow.

#### Where do I set or modify the `drupal_hash_salt` value in Drupal 7?

There can be an occasion when you may need to set the hash salt to a specific value. If you install Drupal 7, it will create a `drupal_hash_salt` value for you, but if you want to use a different one, you can edit `settings.php` before installation. Pantheon uses Pressflow to automatically read the environmental configuration and the Drupal 7 hash salt is stored as part of the Pressflow settings.


    // All Pantheon Environments.
    if (defined('PANTHEON_ENVIRONMENT')) {
      // Set your custom hash salt value.
      $custom_hash_salt = '';
      // Extract Pressflow settings into a php object.
      $pressflow_settings = json_decode($_SERVER['PRESSFLOW_SETTINGS']);
      $pressflow_settings->drupal_hash_salt = $custom_hash_salt;
      $_SERVER['PRESSFLOW_SETTINGS'] = json_encode($pressflow_settings);
     }


#### Where can I get a copy of a default.settings.php?

- Drupal 8 - [https://github.com/pantheon-systems/drops-8/blob/master/sites/default/default.settings.php](https://github.com/pantheon-systems/drops-8/blob/master/sites/default/default.settings.php)
- Drupal 7 -  [https://github.com/pantheon-systems/drops-7/blob/master/sites/default/default.settings.php](https://github.com/pantheon-systems/drops-7/blob/master/sites/default/default.settings.php)
- Drupal 6 -  [https://github.com/pantheon-systems/drops-6/blob/master/sites/default/default.settings.php](https://github.com/pantheon-systems/drops-6/blob/master/sites/default/default.settings.php)

#### Where can I find examples of Pantheon settings.php?
You can view examples at the [pantheon-settings-examples repo](https://github.com/pantheon-systems/pantheon-settings-examples).

#### Are table prefixes supported?

Pantheon injects the database configuration dynamically during bootstrap. In the `PRESSFLOW_SETTINGS` variable, the appropriate database connection information is passed in based upon the environment (Dev/Test/Live).

You can technically use database prefixes, but Pantheon will not support database prefixes. As a best practice, allow Pantheon to populate your database configuration settings.

#### Why is the Status tab showing that my configuration file is not protected and that I need to create a settings.php file?

Drupal doesn't ship with a `settings.php` in place; as the error suggests, you should make a copy of the `default.settings.php` and rename it `settings.php`. Once you have created a `settings.php` file, the `settings.php` area of the report should change to green.

#### Can I edit settings.pantheon.php?
No; `settings.pantheon.php` is for Pantheon's use only and you should only modify the `settings.php` file. The `settings.pantheon.php` file may change in future updates, and modifying it would cause conflicts.

## Troubleshooting
### Request to a Remote API Does Not Return Expected Response

The PHP 5.5 default is `&` and the PHP 5.3 default is `&amp;`.

If the API expects `&` as an argument separator but receives `&amp;` (for example, when using http_build_query), you can override the default arg_separator.ouput value by adding the following line to `settings.php`:

    ini_set('arg_separator.output', '&');

### Drush Error: "No Drupal site found", "Could not find a Drupal settings.php file", or missing system information from status

```bash
Could not find a Drupal settings.php file at ./sites/default/settings.php
```

To resolve, add a default or empty `sites/default/settings.php` to your site's code.

### Error: "The provided host name is not valid for this server."

This error comes from a feature in Drupal 8 designed to protect against [HTTP HOST Header attacks](https://www.drupal.org/node/1992030){.external}. Drupal 8 allows you to specify "trusted host patterns," which specify a set of domains that incoming requests must match.

If you see this error, you need to update your [trusted host patterns](#trusted-host-setting) in `settings.php` and add your new domain(s) to the `$settings['trusted_host_patterns']` array.

