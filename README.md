# freenas-iocage-wordpress
Script to create an iocage jail on FreeNAS for the latest WordPress release, including MariaDB, the PHP scripting language, Caddy, Redis and phpMyAdmin.

This script will create an iocage jail on FreeNAS 11.3 or TrueNAS CORE 12.0 with the latest release of WordPress, along with its dependencies. It will configure the jail to store the database and WordPress data outside the jail, where it can be better managed.

**THIS IS NOT AN INDEPENDENT MODULE. IT MUST BE USED IN CONJUNCTION WITH A REVERSE PROXY. REFER TO THE PREREQUISITES BELOW.**

## Status
This script will work with FreeNAS 11.3, and it should also work with TrueNAS CORE 12.0.  Due to the EOL status of FreeBSD 11.2, it is unlikely to work reliably with earlier releases of FreeNAS.

## Usage

### Prerequisites (Reverse Proxy)
The WordPress jail created by this script is designed to work behind a reverse proxy. If you don't already have a reverse proxy in place, you must set this up first. Do not attempt to run the WordPress setup wizard directly from the jail. I cannot impress the importance of this enough; you must run the WordPress setup wizard using the FQDN for the jail (as configured in the reverse proxy) and not via the jail IP.

Running the WordPress wizard directly from the jail messes with the formatting when later trying to retrofit WordPress behind a reverse proxy. To appreciate this issue, refer to item **3. Problems retrofitting a WordPress jail behind a reverse proxy** in the blog post [WordPress Script: Opportunities for Improvement](https://blog.udance.com.au/2020/09/20/wordpress-script-opportunities-for-improvement/).

**AGAIN, DO NOT ATTEMPT TO SET UP AND USE THIS RESOURCE WITHOUT A REVERSE PROXY. IT MAY BE IMPOSSIBLE LATER TO RETROFIT IT BEHIND A REVERSE PROXY.**

If you need to set up a reverse proxy, there are at least two resources available in the resources section of the FreeNAS forum that you might like to consider:
1. [Reverse Proxy using Caddy (with optional automatic TLS)](https://www.ixsystems.com/community/resources/reverse-proxy-using-caddy-with-optional-automatic-tls.114/)
2. [How to set up an nginx reverse proxy with SSL termination in a jail](https://www.ixsystems.com/community/resources/how-to-set-up-an-nginx-reverse-proxy-with-ssl-termination-in-a-jail.132/)

If you use either resource, please refer any reverse proxy questions you may have to the discussion area for those resources.

### Prerequisites (Other)
Although not required, it's recommended to create a Dataset named `apps` with a sub-dataset named `wordpress` on your main storage pool and nested sub-datasets `files` and `db`.  Many other jail guides also store their configuration and data in subdirectories of `pool/apps/` 

For optimal performance, set the record size of the `db` dataset to 16 KB (under Advanced Settings in the FreeNAS web GUI).  It's also recommended to cache only metadata on the `db` dataset; you can do this by running `zfs set primarycache=metadata poolname/apps/wordpress/db`. 

If these datasets are not present, directories `/apps/wordpress/files` and `/apps/wordpress/db` will be created in `$POOL_PATH`.

### Installation
Download the repository to a convenient directory on your FreeNAS system by changing to that directory and running `git clone https://github.com/basilhendroff/freenas-iocage-wordpress`.  Then change into the new `freenas-iocage-wordpress` directory and create a file called `wordpress-config` with your favorite text editor.  In its minimal form, it would look something like this:
```
JAIL_IP="192.168.1.4"
DEFAULT_GW_IP="192.168.1.1"
TIME_ZONE="Australia/Perth"
```
Many of the options are self-explanatory, and all should be adjusted to suit your needs, but only a few are mandatory.  The mandatory options are:

* JAIL_IP is the IP address for your jail.  You can optionally add the netmask in CIDR notation (e.g., 192.168.1.199/24).  If not specified, the netmask defaults to 24 bits.  Values of less than 8 bits or more than 30 bits are invalid.
* DEFAULT_GW_IP is the address for your default gateway
* TIME_ZONE is the time zone of your location, in PHP notation--see the [PHP manual](http://php.net/manual/en/timezones.php) for a list of all valid time zones.
 
In addition, there are some other options which have sensible defaults, but can be adjusted if needed. These are:

- JAIL_NAME: The name of the jail, defaults to `wordpress`.
- POOL_PATH: The path for your data pool. It is set automatically if left blank.
- FILES_PATH: WordPress site data is stored in this path; defaults to `$POOL_PATH/apps/wordpress/files`.
- DB_PATH: MariaDB database files are stored in this path; defaults to `$POOL_PATH/apps/wordpress/db`.
- INTERFACE: The network interface to use for the jail. Defaults to `vnet0`.
- VNET: Whether to use the iocage virtual network stack. Defaults to `on`.

Organisationally, I've found the following minimal form for `wordpress-config` useful for multiple site, multiple database configurations.
```
JAIL_IP="192.168.1.4"
DEFAULT_GW_IP="192.168.1.1"
TIME_ZONE="Australia/Perth"
JAIL_NAME="site1"
FILES_PATH="/mnt/tank/apps/wordpress/site1/files"
DB_PATH="/mnt/tank/apps/wordpress/site1/db"
```

### Execution
Once you've downloaded the script and prepared the configuration file, run this script `script wordpress.log ./wordpress-jail.sh`.  The script will run for several minutes.  When it finishes, your jail will be created, and WordPress will be installed with all its dependencies. Next, complete the post-installation tasks. 

### Post-Installation Tasks
Refer to post-installation instructions at https://github.com/basilhendroff/freenas-iocage-wordpress/blob/master/POST-INSTALL.md. Alternatively, if you prefer visual cues,  refer to the blog post [WordPress Script: Post-Installation Tasks](https://blog.udance.com.au/2020/09/19/wordpress-script-post-installation-tasks/).

## Support and Discussion
There are self-help resources for WordPress at https://wordpress.org/support/ and support for WordPress, it's themes and plugins in the WordPress support forums at https://wordpress.org/support/forums/.

Questions or issues about this resource can be raised in [this forum thread](https://www.ixsystems.com/community/threads/scripted-wordpress-installation.87604/). Support is limited to getting WordPress up and running in a FreeNAS jail using the script, including the post-installation tasks. 

### Troubleshooting
Having installation or post-installation issues? First things to check are `wordpress.log` and `wordpress-config`. You may be asked to provide these if requesting assistance in the discussion area. 

You may find the logs below useful for troubleshooting. Assuming your jail is named `wordpress`, you can access these through a terminal using `iocage console wordpress`.
1. The Caddy webserver log file `/var/log/caddy.log` and access log `/var/log/access.log`
2. The MariaDB error log for the site `wordpress` at `/var/db/mysql/wordpress.err`
3. The PHP-FPM log file at `/var/log/php-fpm.log`
4. The Redis log file at `/var/log/redis/redis.log`
5. The mail log at `/var/log/maillog`
6. If enabled, the WordPress log file at `/usr/local/www/wordpress/wp-content/debug.log`

### To Do
There are a number of opportunities to continue to improve the script. Much of it is outside my current sphere of knowledge. You can find the outstanding to-do list in the blog post [WordPress Script: Opportunities for Improvement](https://blog.udance.com.au/2020/09/20/wordpress-script-opportunities-for-improvement/). If you're able to assist with any of this, or can help refine the script in other ways, please consider submitting a pull request at https://github.com/basilhendroff/freenas-iocage-wordpress. 

I'd also like to hear of any other suggestions for improving the performance, reliability, or security of the scripted resource in the context of its scope, which is: 

> The assumption is that the local network is trusted so unencrypted (HTTP) communication is considered acceptable on the LAN. External (HTTPS) access to the WordPress service is granted via a reverse proxy.

It's not my intention to expand the resource scope, but to work within it.

## Disclaimer
It's your data. It's your responsibility. This resource is provided as a community service. Use it at your own risk.
