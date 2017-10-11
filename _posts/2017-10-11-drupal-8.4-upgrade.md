---
layout: post
title: Drupal 8.3 to 8.4 upgrade guidance
---

BLT 8.9.6+ is compatible with [Drupal 8.4](https://groups.drupal.org/node/517148). 

That is to say, BLT 8.9.6+ will permit you to install Drupal 8.4 and all of its dependencies (e.g., Drush 9, Lightning 2.2+, Symfony 3, etc.) and BLT's own code will continue to to execute correctly.

However, upgrading from Drupal 8.3 to 8.4 requires a number of manual update steps to make your application and custom theme compatible with the new version of core.  

There are three main reasons for this:
* Drupal 8.4 requires new major versions of important packages like Symphony. This is incompatible with some contributed modules (e.g., entity_pilot).
* <del>Drupal 8.4 requires Drush 9 (technically [Drush 8 + Drupal 8.4 works but is unsupported](https://www.drupal.org/node/2874827))</del>
 **As of Drush 8.1.15, Drush 8 can be used with Drupal 8.4.** Drush 9 itself is only in beta, deprecates many common commands, and [does not support importing configuration during installation](https://github.com/drush-ops/drush/issues/2953) (unless you're using the minimal profile). This creates a tricky situation.
* Drupal 8.4 introduces many changes to core's JavaScript packages. Notably, the new requirement for eslint is incompatible with [Cog](https://github.com/acquia-pso/cog)'s node JS dependencies. jQuery has also been updated from version 2 to version 3. This may break your theme.

# Instructions

Start with a fresh database either via `blt sync` or `blt setup`:

    blt setup

Update your root composer.json file to support Asset Packagist (if you have not already upgraded to Lightning 2.1.8)

    cd docroot/profiles/contrib/lightning
    composer run enable-asset-packagist

OR

Manually update your composer.json to include the asset repository

    "asset-packagist": {
            "type": "composer",
            "url": "https://asset-packagist.org"
        }

 

## Update dependencies

Update dependencies to use latest versions of Drupal core and Lightning. _Warning: these updates may also require updating Behat and the Drupal Extension to the most current versions to avoid Symfony collisions._

It is recommended that you use Drush 8, but it is possible to use Drush 9.

### Drush 8 (recommended)

    composer require acquia/blt:^8.9.7 acquia/lightning:^2.2.0 drupal/core:8.4.0 drush/drush:8.x-dev --update-with-dependencies

### Drush 9 (beta)

    composer require acquia/blt:^9.0.0-alpha1 acquia/lightning:^2.2.0 drupal/core:8.4.0 --update-with-dependencies

Drush 9 removes the drush launcher, so this should be manually installed (otherwise Drush will no longer work on a per project basis (see https://github.com/drush-ops/drush-launcher). 

OSX:

        curl -OL https://github.com/drush-ops/drush-launcher/releases/download/0.4.2/drush.phar

Linux:

        wget -O drush.phar https://github.com/drush-ops/drush-launcher/releases/download/0.4.2/drush.phar

Make downloaded file executable: 

        chmod +x drush.phar

Move drush.phar to a location listed in your $PATH, rename to drush:

        sudo mv drush.phar /usr/local/bin/drush

Almost all BLT builds import configuration during site install. To make this continue working with Drush 9, you must make the following modifications to your application.

Add [core patch](https://www.drupal.org/files/issues/2788777-91.patch) to [support configuration import from profiles](khttp://drupal.org/node/2788777) (this is manual, there is no command):

        "patches": {
            "drupal/core": {
                "Allow a profile to be installed from existing config": "https://www.drupal.org/files/issues/2788777-91.patch"
            }
         }

Modify your custom profile's [profile-name].info.yml file, indicating that configuration should be imported during profile installation:

    ./vendor/bin/yaml-cli update:value docroot/profiles/custom/[profile-name]/[profile-name].info.yml config_install true

Modify BLTs project.yml, indicating that the `--config-dir` option should not be used during `drush site-install`.

    ./vendor/bin/yaml-cli update:value blt/project.yml setup.drupal.install.import-config false

Warning: yaml-cli often writes the boolean false as a string ("false"). Confirm that it is listed correctly as a boolean in both the project.yml and [profile-name].info.yml files!

If you are using features, part of features is incompatible with Drush 9. Update project.yml to work around:

    cm.features.no-overrides: false

### Database updates and config export

Execute database updates and export modified configuration to disk.

    drush updb
    drush pm-uninstall bartik
    drush cex -y

Re-install Drupal and re-export configuration, as the new version of core may export configuration differently. Warning: it is likely that Drupal's installation / configuration install process will be interrupted due to fatal errors. Export the configuration as instructed and try again.
    
    blt setup
    drush cr
    drush cex -y

Commit your changes and test that everything works as expected.

    git add -A
    git commit -m "Updating from Drupal 8.3.x to 8.4.0."
    blt setup
    blt tests

# Cog build process workaround

Drupal 8.4 adds new JavaScript packages that are not compatible with previous versions of Cog. If you're using a cog based theme, you must add new node dependencies to it and update some of your scripts to use those new dependencies.

`cd` into your custom theme directory and execute the following:

    npm install --save-dev eslint-config-airbnb@14.1.0 eslint-plugin-import@^2.2.0 eslint-plugin-react@6.10.3 eslint@3.19.0 eslint-plugin-jsx-a11y@4.0.0

Next, you must modify a few scripts in your custom theme. The purpose of this is to avoid namespace collisions between the `gulp-eslint` and the `eslint` packages.  The following files must be modified:

* gulp-tasks/lint-js.js
* .eslintignore
* gulpfile.js

Please see [this change set](https://gist.github.com/grasmash/09a3cb608fa4be8f40494f4abd8537e6) in Canary for specific line changes.
