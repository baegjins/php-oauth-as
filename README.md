[![Build Status](https://travis-ci.org/fkooman/php-oauth-as.png?branch=master)](https://travis-ci.org/fkooman/php-oauth-as)

# Introduction
This is an OAuth 2.0 Authorization Server written in PHP that is easy to 
integrate with your existing REST services, written in any language. It will
require minimal changes to your existing software.

# License
Licensed under the GNU Affero General Public License as published by the Free 
Software Foundation, either version 3 of the License, or (at your option) any 
later version.

    https://www.gnu.org/licenses/agpl.html

This roughly means that if you use this software in your service you need to 
make the source code available to the users of your service (if you modify
it). Refer to the license for the exact details.

# Features
* PDO database backend support
* Authorization Code and Implicit Grant support
* `BasicAuthenication` Backend (configured in configuration file)
* `MellonAuthentication` Backend (SAML)
* Token Introspection for Resource Servers
* Management API to manage Client Registration and Authorizations

# Screenshots
This is a screenshot of the consent dialog:

![oauth_consent](https://github.com/fkooman/php-oauth-as/raw/master/docs/oauth_consent.png)

# Installation
Please use the RPM packages for actually running it on a server. The RPM 
packages can for now be found in the 
[repository](https://copr.fedoraproject.org/coprs/fkooman/php-oauth/). For 
setting up a development environment, see below.

Currently the documentation for getting started with the RPM packages still 
needs to be written...

# Development Requirements
On Fedora/CentOS:

    $ sudo yum install php-pdo php-openssl httpd'

You also need to download [Composer](https://getcomposer.org/).

The software is being tested with Fedora 20, CentOS 6 and CentOS 7 and should 
also work on RHEL 6 and RHEL 7.

# Development Installation
*NOTE*: in the `chown` line you need to use your own user account name!

    $ cd /var/www
    $ sudo mkdir php-oauth-as
    $ sudo chown fkooman.fkooman php-oauth-as
    $ git clone https://github.com/fkooman/php-oauth-as.git
    $ cd php-oauth-as
    $ /path/to/composer.phar install
    $ mkdir data
    $ sudo chown apache.apache data
    $ sudo semanage fcontext -a -t httpd_sys_rw_content_t '/var/www/php-oauth-as/data(/.*)?'
    $ sudo restorecon -R /var/www/php-oauth-as/data
    $ cd config
    $ cp oauth.ini.defaults oauth.ini

Edit `oauth.ini` to match the configuration. You need to at least modify the
following lines, and set them to the values shown here:

    entitlementsFile = "/var/www/php-oauth-as/config/entitlements.json"
    dsn = "sqlite:/var/www/php-oauth-as/data/db.sqlite"

Now continue with the configuration:

    $ cp entitlements.json.example entitlements.json

You can modify the `entitlements.json` file to list your own user ID to 
specify the `manage` entitlement. By default the `admin` user ID has this
entitlement.

    $ sudo -u apache bin/php-oauth-as-initdb 
    $ sudo -u apache bin/php-oauth-as-register https://www.php-oauth.net/app/config.json

Copy paste the contents of the Apache section (see below) in the file 
`/etc/httpd/conf.d/php-oauth-as.conf`.

    $ sudo service httpd restart

If you ever remove the software, you can also remove the SELinux context:

    $ sudo semanage fcontext -d -t httpd_sys_rw_content_t '/var/www/php-oauth-as/data(/.*)?'

# Apache
This is the Apache configuration you use for development. Place it in 
`/etc/httpd/conf.d/php-oauth-as.conf` and don't forget to restart Apache:

    Alias /php-oauth-as /var/www/php-oauth-as/web

    <Directory /var/www/php-oauth-as/web>
        AllowOverride None
        Options FollowSymLinks

        <IfModule mod_authz_core.c>
          # Apache 2.4
          Require local
        </IfModule>
        <IfModule !mod_authz_core.c>
          # Apache 2.2
          Order Deny,Allow
          Deny from All
          Allow from 127.0.0.1
          Allow from ::1
        </IfModule>
        
        # CORS
        <FilesMatch "api.php">
            Header set Access-Control-Allow-Origin "*"
            Header set Access-Control-Allow-Headers "Authorization, Content-Type"
            Header set Access-Control-Allow-Methods "POST, PUT, GET, DELETE, OPTIONS"
        </FilesMatch>

        <FilesMatch "introspect.php">
            Header set Access-Control-Allow-Origin "*"
            Header set Access-Control-Allow-Methods "GET, OPTIONS"
        </FilesMatch>

        <FilesMatch "authorize.php">
            # CSP: https://developer.mozilla.org/en-US/docs/Security/CSP
            Header set Content-Security-Policy "default-src 'self'"

            # X-Frame-Options: https://developer.mozilla.org/en-US/docs/HTTP/X-Frame-Options
            Header set X-Frame-Options DENY
        </FilesMatch>

        RewriteEngine On
        RewriteCond %{HTTP:Authorization} ^(.+)$
        RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]

        # HSTS: https://developer.mozilla.org/en-US/docs/Security/HTTP_Strict_Transport_Security
        #Header set Strict-Transport-Security max-age=604800
    </Directory>

Restart Apache with `service httpd restart`.

# Authentication
There are currently two plugins provided for user authentication:

* `BasicAuthentication` - Simple static username/password authentication 
  configured through `config/oauth.ini` (**DEFAULT**)
* `MellonAuthentication` - Plugin for SAML authentication

You can configure which plugin to use by modifying the 
`authenticationPlugin` setting in `config/oauth.ini`.

You do need to configure these authentication backend separately. See the 
respective documentation for those projects.

## Entitlements
This OAuth server also uses *entitlements*. Entitlements are certain access 
rights a particular user using an API has. For instance, you can configure a
user to have `manage` rights on the API of the Authorization Server. 

These entitlements can be provided through a static configuration file 
in `config/entitlements.json`. 

Entitlements are needed because sometimes you want users to be able to use the
provided API, but not be able to manage client registration, or on some APIs, 
some users have more rights. For instance, a special user can access data for
all users through the API while regular users can only access their own data.

The API for managing the authorization server supports the 
`http://php-oauth.net/entitlement/manage` entitlement in order to be able to 
modify application registrations. If for instance the authentication backend 
supports the users `admin`, `fkooman` and `demo` and you want to give the user
with the ID `admin` the entitlement to manage the application registrations, 
you would put that in `config/entitlements.json`:

    {
        "admin": [
            "http://php-oauth.net/entitlement/manage"
        ]
    }

Now, whenever the `admin` user successfully authenticates it can manage clients
through the API. Users with other IDs will not be able to manage the clients.

# Management Clients
There are two management clients available:

* [Manage Applications](https://github.com/fkooman/html-manage-applications/)
* [Manage Authorizations](https://github.com/fkooman/html-manage-authorizations/)

These clients are written in HTML, CSS and JavaScript only and can be hosted on 
any (static) web server. See the accompanying READMEs for more information.
For your convenience they are hosted on 
[https://www.php-oauth.net](https://www.php-oauth.net) so you do not need to 
setup the applications yourself and can immediately use the hosted versions. 
Just specify the endpoint to your Authorization Server to get started. They 
also work with the Docker image, you can then use 
`https://localhost/php-oauth-as/` as the URL to connect to.

# Resource Servers
If you are writing a resource server (RS) an API is available to verify the 
`Bearer` token you receive from the client. Currently a draft specification
`draft-richer-oauth-introspection` is implemented to support this.

An example, the RS gets the following `Authorization` header from the client:

    Authorization: Bearer 40da7666a9f76b4b6b87969a7cc06421

Now in order to verify it, the RS can send a request to the OAuth service:

    $ curl -k -s https://localhost/php-oauth-as/introspect.php?token=40da7666a9f76b4b6b87969a7cc06421 | python -mjson.tool
    {
        "active": true,
        "client_id": "2352ea44-612d-448b-be10-6e29562e5130",
        "exp": 1409564548,
        "iat": 1409560948,
        "scope": "http://php-oauth.net/scope/manage",
        "sub": "admin",
        "token_type": "bearer",
        "x-entitlement": "http://php-oauth.net/entitlement/manage"
        ]
    }
    
The RS can now figure out more about the resource owner. If you provide an 
invalid access token, the following response is returned:

    {
        "active": false
    }

If your service needs to provision a user, the field `sub` SHOULD to be used 
for that. The `scope` field can be used to determine the scope the client was 
granted by the resource owner.

There is a proprietary extensions to this format: `x-entitlement`. It gives the 
entitlement values as a space separated list of entitlements, just like the 
`scope` field.

A plugin for `fkooman/rest`, `fkooman/rest-plugin-bearer` is available to 
integrate with this OAuth 2.0 AS service using 
[Composer](https://getcomposer.org). Or see the project 
[site](https://github.com/fkooman/php-lib-rest-plugin-bearer).
