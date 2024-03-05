---
nav:
  title: Staging
  position: 10

---

# Staging

Since Shopware 6.6.1.0, Shopware has a integrated Staging Mode. This mode prepares the Shop to be used in a staging environment. This means that the shop is prepared to be used in a test environment, where changes can be made without affecting the live shop.

## The workflow

The staging mode is designed, to only modify data inside the Shopware instance. This means, the Staging mode does not duplicate the current Installation, copy Database or Files. It only modifies the data inside the Shopware instance.

So the real world use-case would be kinda like this:

### Creating the second Shopware instance

The recommanded way would to deploy from your Git repository to the new environment. This way, you make sure that the codebase is equal to the live environment. The alternative way would be to copy the files from the live environment to the staging environment.

### Copying the Database

::: info
You should make sure that the `mysqldump` and the `mysql` binary is from the same major version and vendor. This means, if you use `mysqldump` from MariaDB, you should also use `mysql` from MariaDB. The same applies to MySQL.
:::

To have the Staging environment similar as possible to the live environment, it's recommanded to duplicate the database. For this you can use the `mysqldump` command to export the database and import it into the staging environment. 

Our recommandation would be to use `shopware-cli project dump` to create a dump of the database and import it with the regular mysql command. Shopware cli does also have a flag to anonymize the data, so you can be sure that no personal data is in the staging environment.

```bash
# creating a regular dump, the clean parameter will not dump the data of cart table
shopware-cli project dump --clean --host localhost --username db_user --password db_pass --output shop.sql shopware

# create a dump with anonymized data
shopware-cli project dump --clean --anonymize --host localhost --username db_user --password db_pass --output shop.sql shopware
```

You can configure the dump command with a `.shopware-project.yml` which tables should be skipped, which additional fields should be anonymized and more. Checkout the [CLI documentation](https://sw-cli.fos.gg/commands/project/#shopware-cli-project-dump-database) for more information.

### Configuration

::: info
Generally it's not recommanded to share any resources like MySQL, Redis, ElasticSearch/OpenSearch between the live and staging environment. This could lead to data corruption when the configuration is not done correctly. Also the performance of the live environment could be affected by the staging environment.
:::

After the database is imported, you should modify the `.env` to use the staging database. If you use ElasticSearch/OpenSearch, you should set a `SHOPWARE_ES_INDEX_PREFIX` to avoid conflicts with the live environment.

### Activate the Staging Mode

After the database is imported and the configuration is done, you can activate the Staging Mode. This can be done using the command `./bin/console system:setup:staging`. This command will modify the database to be used in a staging environment. You can pass `--no-interaction` to the command to avoid the interactive questions.

## Staging Mode

The Staging Mode is designed to be used in a test environment. This means, that the shop is prepared to be used in a test environment, where changes can be made without affecting the live shop.

### What does the staging mode?

The staging mode does the following:

- Deletes all apps which have an active connection to an external service and the integrations in Shopware
- Reset the instance id used for registration of Apps
- It disables the sending of mails
- Rewrites the URLs to the staging domain (if configured)
- Checks that the ElasticSearch/OpenSearch indices are not existing yet
- Shows a banner in the administration and storefront to indicate that the shop is in staging mode

### What does the staging mode not?

The staging mode does not:

- Duplicate the current Installation
- Copy Database or Files
- Modify the live environment

### Configuration

The staging mode is fully configureable with a `config/packages/staging.yaml`. You can configure the following options:

```yaml
# <shopware-root>/config/packages/staging.yaml
shopware:
    staging:
        mailing:
            # Disables the sending of mails (default: true)
            disable_delivery: true
        storefront:
            # Shows a banner in the storefront when staging mode is active (default: true)
            show_banner: true
        administration:
            # Shows a banner in the administration when staging mode is active (default: true)
            show_banner: true
        sales_channel:
            domain_rewrite:
                # See below for more information
        elasticsearch:
            # Checks that no indices are existing yet (default: true)
            check_for_existence: true
```

One of the most important options is the `domain_rewrite`. This option allows you to rewrite the URLs to the staging domain. This allows multiple ways to rewrite the URLs:

Using direct match (`equal`)

```yaml
# <shopware-root>/config/packages/staging.yaml
shopware:
    staging:
        sales_channel:
            domain_rewrite:
                - type: equal
                  match: https://my-live-store.com
                  replace: https://my-staging-store.com
                - # ... second rule
```

This goes and compares the Sales Channel URL and when it's equal to `https://my-live-store.com` it will be replaced with `https://my-staging-store.com`.

Replace using prefix (`prefix`)

```yaml
# <shopware-root>/config/packages/staging.yaml
shopware:
    staging:
        sales_channel:
            domain_rewrite:
                - type: prefix
                  match: https://my-live-store.com
                  replace: https://my-staging-store.com
                - # ... second rule
```

The difference here to the `equal` type is, that it will only replace the URL when it starts with `https://my-live-store.com`, so all paths to that beginning will be replaced.

`https://my-live-store.com/en` will be replaced with `https://my-staging-store.com/en`

Replace using regex (`regex`)

```yaml
# <shopware-root>/config/packages/staging.yaml
shopware:
    staging:
        sales_channel:
            domain_rewrite:
                - type: regex
                  match: '/https?:\/\/(\w+)\.(\w+)$/m'
                  replace: 'http://$1-$2.local'
                - # ... second rule
```

This will use the regex to replace the URL. The match and replace are regular expressions. In this example a `https://my-live-store.com` will be replaced with `http://my-live-store.local`.

### Usage of Apps

The staging command will delete all apps which have an active connection to an external service and the integrations in Shopware. This means, that the apps are not available in the staging environment. This is done to avoid any data corruption or data leaks to the live environment. After the execution of the command, you can install the apps again in the staging environment. This will not affect the live environment, as the instance id is different.

## Integration into Plugins

The `system:setup:staging` is dispatching a Event which all plugins can subscribe to `Shopware\Core\Maintenance\Staging\Event\SetupStagingEvent` and modify the database for them to be in staging mode.

Example of a subscriber for a payment provider to turn on the test mode:

```php
<?php

namespace Swag\PaymentProvider\Subscriber;

use Shopware\Core\Maintenance\Staging\Event\SetupStagingEvent;

class StagingSubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [
            SetupStagingEvent::class => 'onSetupStaging'
        ];
    }

    public function onSetupStaging(SetupStagingEvent $event): void
    {
        // modify the database to turn on the test mode
    }
}
```
