
#  Eduroam tools container-based deployment: Overall considerations #

The ancilliary tools package consists of three separate sets of tools:
* admintool
* metrics
* monitoring

Each of the tools is (at the moment) designed to run in an isolated environment.  They can be run on a single docker host by mapping each to a different port.  The configuration files provided here are designed this way:

* Admintool runs on ports 80 and 443 (HTTP and HTTPS)
* Monitoring tools run on ports 8080 and 8443 (HTTP and HTTPS)
* Metrics runs on ports 9080 and 9443 (HTTP and HTTPS)

It is possible (and recommended if the resources are available) to run these on separate hosts - in which case each can run on the standard port HTTP/HTTPS ports 80/443. Details on the configuration change needed are given below.

# ChangeLog

Changes to this document since the workshop at APAN41 in Manilla, January 2016.

* 2016-04-05: Added documentation on creating local accounts.
* 2016-04-05: Added section documenting Network access requirements.
* 2016-04-05: Added documentation for adjusting port numbers.
* 2016-04-05: Added documentation for installing proper SSL certificates.
* 2016-04-05: Added Metrics tools authentication configuration.
* 2016-04-04: Added Admintool settings for NREN Institution and Federation name.
* 2016-04-01: Added instructions for loading Icinga configuration generated by Admintool.
* 2016-04-01: Added Update instructions.
* 2016-04-01: Added this ChangeLog section.
* 2016-03-08: Added documentation for passing arbitrary configuration options to Admintool.
* 2016-03-08: Added configurable login methods for Admintool.
* 2016-02-25: Added ADMINTOOL_SECRET_KEY environment variable.
* 2016-02-15: Added ADMINTOOL_DEBUG environment variable.
* 2016-02-15: Clarified deployment instructions.
* 2016-02-15: Added missing instructions to enable Google+API.

# Preliminaries - Docker

Install and configure Docker.  Please follow our [Docker setup instructions](Docker-setup.md).

Please become familer with Docker by following our [Docker introduction](Docker-intro.md).

# Preliminaries - Mail server

Some of the tools (admintool and monitoring) will need to send outgoing email.  Please make sure you have the details of an SMTP ready - either one provided by your systems administrator, or one running on the local system.

# Eduroam ancillary tools: Basic setup

NOTE: For the XeAP workshop at APAN41, please follow instructions in [Lab.md](Lab.md)

The installation instructions here are ment for deployment at your institution - the ones in [Lab.md](Lab.md) are targeted for the lab VMs.

On each of the VMs, start by cloning the git repository:

    git clone https://github.com/REANNZ/etcbd-public

# Deploying admintool

Modify the ````admintool.env```` file with deployment parameters - override at least the following values:

* Pick your own admin password: ````ADMIN_PASSWORD````
* Pick internal DB passwords: ````DB_PASSWORD````, ````POSTGRES_PASSWORD````
  * Generate these with: ````openssl rand -base64 12````
* ````SITE_PUBLIC_HOSTNAME````: the hostname this site will be visible as.
* ````LOGSTASH_HOST````: the hostname the metrics tools will be visible as.
* ````EMAIL_*```` settings to match the local environment (server name, port, TLS and authentication settings)
* ````SERVER_EMAIL````: From: email address to use in outgoing notifications.
* ````ADMIN_EMAIL````: where to send NRO admin notifications.
* ````REALM_COUNTRY_CODE```` / ````REALM_COUNTRY_NAME```` - your eduroam country
* ````NRO_INST_NAME````: the name of the institution acting as the National Roaming Operator (NRO)
* ````NRO_FEDERATION_NAME````: the name of the AAI federation the Admintool is connected to (if available to the NRO).  Leave unmodified if no AAI federation exists locally.
* ````TIME_ZONE```` - your local timezone (for the DjNRO web application)
* ````MAP_CENTER_LAT````, ````MAP_CENTER_LONG```` - pick your location
* ````REALM_EXISTING_DATA_URL````: In a real deployment, set this to blank (````REALM_EXISTING_DATA_URL=````) to start with an empty database.  Leaving it with the value provided would import REANNZ data - suitable for a test environment.
* ````GOOGLE_KEY````/````GOOGLE_SECRET```` - provide Key + corresponding secret for an API credential (see below on configuring these settings)
* ````ADMINTOOL_SECRET_KEY````: this should be a long and random string used internall by the admintool.
  * Please generate this value with: ````openssl rand -base64 48````
* ````ADMINTOOL_DEBUG````: for troubleshooting, uncomment the existing line: ````ADMINTOOL_DEBUG=True```` - but remember to comment it out again (or change to False or blank) after done with the troubleshooting.
* ````ADMINTOOL_LOGIN_METHODS````: enter a space-separated list of login methods to enable.
  * Choose from:
    * shibboleth: SAML Login with Shibboleth SP via an identity federation.  Not supported yet.
    * locallogin: Local accounts on the admin tool instance.
       * Note that local accounts can later be created by logging into the
         Admintool at https://admin.example.org/admin/ as the administrator
         (with the username and password created here), and selecting `Users`
         from the list of tables to administer, and creating the user with the `Add user` button.
    * google-oauth2: Login with a Google account.  Only works for applications registered with Google - see below on enabling Google login.
    * google-plus: Login with a Google account via Google Plus.  May be used as an alternative to Google OAuth2.  Also only works for applications registered with Google - see below on enabling Google login.
    * yahoo: Login with a Yahoo account.  No registration needed.
    * amazon: Login with an Amazon account.  Registration needed.
    * docker: Login with a Docker account.  Registration needed.
    * dropbox-oauth2: Login with a Dropbox account.  Registration needed.
    * facebook: Login with a Facebook account.  Registration needed.
    * launchpad: Login with an UbuntuOne/Launchpad account.  No registration needed.
    * linkedin-oatuh2: Login with a LinkedIn account.  Registration needed.
    * meetup: Login with a MeetUp account.  Registration needed.
    * twitter: Login with a Twitter account.  Registration needed.
  * Please note that many of these login methods require registration with the target site, and also need configuring the API key and secret received as part of the registration.  Please see the [Python Social Auth Backends documentation](https://python-social-auth.readthedocs.org/en/latest/backends/) for the exact settings required.
* Additional settings: it is also possible to pass any arbitrary settings for the Admintool (or its underlying module) by prefixing the setting with ````ADMINTOOL_EXTRA_SETTINGS_````.  This would be relevant for passing configuration entries to the login methods configured above - especially SECRET and KEY settings for login methods requiring these.  Example: to pass settings required by [Twitter](https://python-social-auth.readthedocs.org/en/latest/backends/twitter.html), add them as:

        ADMINTOOL_EXTRA_SETTINGS_SOCIAL_AUTH_TWITTER_KEY=93randomClientId
        ADMINTOOL_EXTRA_SETTINGS_SOCIAL_AUTH_TWITTER_SECRET=ev3nM0r3S3cr3tK3y

* Icinga configuration: please see the separate section on using the Admintool to generate configuration for Icinga.

Note this file (````admintool.env````) is used both by containers to populate runtime configuration and by a one-off script to populate the database.

As an additional step, in ````global-env.env````, customize system-level ````TZ```` and ````LANG```` as desired.

Use Docker-compose to start the containers:

    cd etcbd-public/admintool
    docker-compose up -d

Run the setup script:

    ./admintool-setup.sh admintool.env

At this point, please become familiar with Docker-compose by following our [Introduction to Docker-compose](Docker-compose-intro.md):

## Installing proper SSL certificates for Admintool

We strongly recommend operating the Admintool with proper SSL certificates.  The tool would generate a self-signed certificate on startup in order to operate, but this would make interaction difficult for both end-users (browser access) as well as for automated tools.

We leave obtaining the certificates outside the scope of this document.
* Obtain the certificates in PEM format.
* Combine the server certificate with any intermediate certificates (concatenated into a single file) and store this as `server.crt` in a directory on the server running the Admintool.
* Store the private key as `server.key` in the same directory.
* Copy the two files into the Apache certificates volume with:

        docker run -v `pwd`:/certs-in -v admintool_apache-certs:/apache-certs -it --rm --name debcp debian:jessie cp /certs-in/server.crt /certs-in/server.key /apache-certs

## Preparing Icinga configuration in Admintool

In the next section, we will be deploying the monitoring tools (Icinga).
The monitoring tools will be loading the configuration generated by the
Admintool.  We need to configure this before we proceed to installing the
monitoring tools.

The generated configuration will:
* Define an Icinga Host for each NRO Radius server.
* Define an Icinga Host for each Institutional Radius server.
* Configure a Ping connectivity check for each server.
* If supported by the Radius server, generate a Radius Status Check for the server.
* For each Monitored Realm with authentication configured, there will be:
  * A service check through each NRO server.
  * A service check through each Institutional Radius server that is listed as a proxy for the Institutional Realm.
* Each service check will be configured with Notifications to Contacts associated with Institution the Institutional Realm being checked belongs to.
* For each Institutional Radius server, the Notifications would also go to all Contacts associated with all Institutions the server is associated with.

There is a number of configuration variables in `admintool.env` that control how the configuration will be generated:
* `NRO_SERVERS` specifies the list of NRO servers (short identifiers) - example: `NRO_SERVERS=server1 server2`
* For each server, there should be a set of variables using the short server identifier as part of the name:
  * `NRO_SERVER_HOSTNAME_serverid`: the hostname (or IP address) to use for server checks
  * `NRO_SERVER_SECRET_servererid`: the Radius secret to use with the server
  * `NRO_SERVER_PORT_serverid`: the Radius port number to use with this server (Optional, defaults to 1812)
  * `NRO_SERVER_STATUS_serverid`: should be set to `True` if the server supports Radius Status checks (Optional, defaults to False)
* `ICINGA_CONF_REQUEST_CUI`: should be set to `True` if the eduroam login checks should request the Chargeable User Identity (CUI) attribute (Optiona, defaults to False)
* `ICINGA_CONF_OPERATOR_NAME`: the Operator Name to use in the eduroam login checks (Optional, no operator name is passed if not specified).
* `ICINGA_CONF_VERBOSITY`: the verbosity level in eduroam login checks.  (Optional, defaults to 1.  Level of at least 1 is needed to see the CUI returned by the check.  Values can range from 0 to 2).
* `ICINGA_CONF_GENERATE_INSTSERVER_CHECKS`: Should the generated configuration include the institutional server checks? (Optiona, defaults to True).  This setting gives the option to disable the institional server checks if not desired.
* `ICINGA_CONF_NOTIFY_INST_CONTACTS`: Should the generated configuration notify institutional contacts - for server and monitored realm checks? (Optiona, defaults to True).  This setting gives the option to disable all Notifications sent to institional contacts if not desired - in which case alerts would go only to the nominated NRO email address given in the Monitoring tools configuration.

* Update Admintool to pick up the changed configuration:

        docker-compose up -d

As a final step, prepare the account Icinga would use to create the configuration:

* Log into the Admintool at https://admin.example.org/admin/ as the administrator (with the username and password created earlier).
* Select `Users` from the list of tables to administer.
* Use the `Add user` button to bring up the user creation form.
* Enter the username and password
  * The sample Monitoring configuration uses `confuser` as the username
  * We recommend generating the password, e.g. with `openssl rand -base64 12``
* Use the `Save and continue editing` button to get to the next screen with additional details
  * In the list of `Available user permissions`, select all three `edumanage | Monitored Realm (local authn)` permissions and add them to the `Chosen user permissions`.  (The permission to access the monitoring credentials is internally used to represent as permission to access the monitoring configuration).

Now you should be able to access the monitoring configuration at https://admin.example.org/icingaconf with this account.  (And it is also accessible under the administrator account).

# Deploying monitoring tools

Modify the ````icinga.env```` file with deployment parameters - override at least the following values:

* Pick your own admin password: ````ICINGAWEB2_ADMIN_PASSWORD````
* Pick internal DB passwords: ````ICINGA_DB_PASSWORD````, ````ICINGAWEB2_DB_PASSWORD````, ````POSTGRES_PASSWORD````
  * Generate these with: ````openssl rand -base64 12````
* ````SITE_PUBLIC_HOSTNAME````: the hostname this site will be visible as
* ````ICINGA_ADMIN_EMAIL````: where to send notifications
* ````EMAIL_*```` settings to match the local environment (server name, port, TLS and authentication settings)
* ````EMAIL_FROM```` - From: address to use in outgoing notification emails.
* The following configure Icinga monitoring and check and notification intervals:
  * `ICINGA_NOTIFICATION_INTERVAL`: Re-notification interval in seconds.  0 disables re-notification.  Defaults to 7200 (2 hours).
  * `ICINGA_CHECK_INTERVAL`: Check interval in seconds (HARD state).  Defaults to 300 (5 minutes)
  * `ICINGA_RECHECK_INTERVAL`: Interval (in seconds) for rechecking after a state change (SOFT state).  Defaults to 60 seconds.
* The following settings configure how Icinga fetches the configuration generated by the Admintool:
 * `CONF_URL_LIST`: the URL (possibly a list of URLs) to fetch configuration from.  Should be https://eduroam-admin.example.org/icingaconf
 * `CONF_URL_USER`: the username to use to authenticate to the configuration URL.
 * `CONF_URL_PASSWORD`: the password to use to authenticate to the configuration URL.
 * `CONF_REFRESH_INTERVAL`: the time period (in seconds) to wait before reloading the configuration from the URL.  Defaults to 3600 (1 hour).
 * `WGET_EXTRA_OPTS`: additional options to passs to `wget` when fetcing the configuration.  This needs in particular to take care of `wget` establishing trust for the certificate presented by the server.
   * If the Admintool Apache server has already been configured with a certificate from an accredited Certification Authority (optional step above), no further action is needed.
   * If the Admintool is using a self-signed automatically generated certificate, we recommend:
     * Copying this certificate into the Icinga configuration volume so that `wget` can access the certificate:
       * Note that at this stage, the volume for the Icinga external configuration has not been created yet, so we need to jump the gun and create it explicitly (otherwise, the volumes get created automatically by `docker-compose up`): `docker volume create --name icinga_icinga-external-conf`
       * Copy the certficiate (by running the "cp" command in a container mounting both volumes): `docker run --rm --name debian-cp -v admintool_apache-certs:/admintool-apache-certs -v icinga_icinga-external-conf:/icinga_icinga-external-conf debian:jessie cp /admintool-apache-certs/server.crt /icinga_icinga-external-conf/`
     * And instructing `wget` to trust this certificate: `WGET_EXTRA_OPTS=--ca-certificate=/etc/icinga2/externalconf/server.crt`
     * Alternatively, it is also possibly to instruct `wget` to blindly accept any certificate - but as this creates serious security risks, it can only be used for internal tests and MUST NOT be used in production.  The setting is: `WGET_EXTRA_OPTS=--no-check-certificate`


This file is used by both the containers to populate runtime configuration and by a one-off script to populate the database.

Additionally, in ````global-env.env````, customize system-level ````TZ```` and ````LANG```` as preferred - or you can copy over global.env from admintool:

    cd etcbd-public/icinga
    cp ../admintoool/global-env.env .

Use Docker-compose to start the containers:

    cd etcbd-public/icinga
    docker-compose up -d

Run the setup script:

    ./icinga-setup.sh icinga.env

Optional: Install proper SSL certificates into /var/lib/docker/host-volumes/icinga-apache-certs/server.{crt,key}

Note: at this point, Icinga will be executing checks against all Radius servers as configured.  It is essential to also configure the Radius servers to accept the Icinga host as a Radius client - with the secret as configured in the Admintool.

## Installing proper SSL certificates for Monitoring tools

We also recommend operating the Monitoring tools with proper SSL certificates.

We leave obtaining the certificates outside the scope of this document.
* Obtain the certificates in PEM format.
* Combine the server certificate with any intermediate certificates (concatenated into a single file) and store this as `server.crt` in a directory on the server running the Monitoring tools.
* Store the private key as `server.key` in the same directory.
* Copy the two files into the Apache certificates volume with:

        docker run -v `pwd`:/certs-in -v icinga_icingaweb-apache-certs:/apache-certs -it --rm --name debcp debian:jessie cp /certs-in/server.crt /certs-in/server.key /apache-certs

# Deploying metrics tools

Modify the ````elk.env```` file with deployment parameters - override at least the following values:

* Pick your own admin password: ````ADMIN_PASSWORD````
* ````ADMIN_USERNAME````: optionally configure the administrator username (default: `admin`)
* ````SITE_PUBLIC_HOSTNAME````: the hostname this site will be visible as.
* ````ADMIN_EMAIL````: email address so far only used in web server error messages.

Additionally, in ````global-env.env````, customize system-level ````TZ```` and ````LANG```` as preferred - or you can copy over global.env from admintool:

    cd etcbd-public/elk
    cp ../admintoool/global-env.env .

Use Docker-compose to start the containers:

    docker-compose up -d

## Installing proper SSL certificates for Metrics tools

We also recommend operating the Metrics tools with proper SSL certificates.

We leave obtaining the certificates outside the scope of this document.
* Obtain the certificates in PEM format.
* Combine the server certificate with any intermediate certificates (concatenated into a single file) and store this as `server.crt` in a directory on the server running the Metrics tools.
* Store the private key as `server.key` in the same directory.
* Copy the two files into the Apache certificates volume with:

        docker run -v `pwd`:/certs-in -v elk_apache-certs:/apache-certs -it --rm --name debcp debian:jessie cp /certs-in/server.crt /certs-in/server.key /apache-certs

# Accessing the services

The Admintool can be accessed at https://admin.example.org/

The management interface of the Admnintool can be accessed at https://admin.example.org/admin/

The management interface of the monitoring tools (Icingaweb2) can be accessed at https://monitoring.example.org:8443/icingaweb2/

The web interface of the metrics tools (Kibana) can be accessed at https://metrics.example.org:9443/icingaweb2/

# Configuring port numbers

The configuration for the three services has been done to allow them to run on
the same host (avoiding port clashes) - but that means Monitoring and Metrics
web services run on nonstandard port numbers.

If the tools are deployed on separate hosts, it would be desirable to run them
on standard port numbers (as there is no longer the need to avoid port
clashes).

To make the Monitoring tools run on standard port numbers: edit
`icinga/docker-compose.yml` and change the port mappings for the `icingaweb`
service from:

    ports:
      # temporarily use alt ports to avoid clash with admintool
      - "8080:80"
      - "8443:443"

to:

    ports:
      - "80:80"
      - "443:443"

and also change the `HTTPS_PORT` variable to match the new port number - i.e., from:

    environment:
        HTTPS_PORT: 8443

to:

    environment:
        HTTPS_PORT: 443

Restart the containers with:

    cd etcbc-public/monitoring
    docker-compose up -d

To make the Metrics tools run on standard port numbers, edit
`elk/docker-compose.yml` and change the port mappings for the `apache` service
and the `HTTPS_PORT` environment variable the same way (and also restart the
containers.

# Network access requirements

The Admintool, Monitoring, and Metrics tools provide a variety of services with
different audiences - the following lists access that should be granted:

* The Admintool is to be accessed by NRO staff, institutional administrators,
  and the general public - so should be publicly accessible on TCP ports 80 and 443.
* The Monitoring and Metrics tools would be accessed primarily by NRO staff
  over HTTP/HTTPS.  Access is protected by username/password authentication
  over HTTPS - so depending on local policies and preferences, the HTTP+HTTPS
  ports can be made accessible either publicly, or just from the NRO internal network.
* The Metrics tools also listen on TCP port 5043 for incoming messages from
  other services (primarily the Radius server, also Apache on the Admintool).
  This port should be made accessible only to the services pushing in data and
  MUST NOT be publicly accessible (i.e., MUST be protected by a firewall).

# Updating deployed tools

Over time, these tools will receive updates - as upstream software releases new
version, as new features are developed, or as bugs and security issues are
fixed.

The updates would target both the container images and the files driving the
images contained in these repositories.

It is essential to install these updates to continue operating these tools in a reliable and secure manner.

## Updating Docker files

The first step is to pull updates to the Docker (and docker-compose) files driving the tools - i.e., this repository:

    cd etcbd-public
    git fetch --verbose --all
    git merge origin/master

If the above commands succeed, this part is completed.  Git may fail with an error message like:

````
error: Your local changes to the following files would be overwritten by merge:
        admintool/admintool.env
        Please, commit your changes or stash them before you can merge.
        Aborting
````

In that case:

* Save your changes to the file(s) affected into the git "stash" with: `git stash save`
* Now merge the updates to the original (unmodified) files: `git merge origin/master`
* Now bring back the changes from the stash: `git stash pop`
* Git tries to merge the local modifications into the updated files.
* If this succeeds without encountering any conflicts, this step is done.
* If git reports conflicts - like:

        $ git stash pop
        Auto-merging admintool/admintool.env
        CONFLICT (content): Merge conflict in admintool/admintool.env

* It is necessary to manually resolve the conflict.  Edit the file and manually merge the updates (typically adding new settings) with local modifications (customizations of existing settings).  The file would contain fragments from both versions, separated by clearly visible demarcation lines:

        <<<<<<< Updated upstream
        #ADMINTOOL_DEBUG=True
        ADMINTOOL_LOGIN_METHODS=google-oauth2 launchpad yahoo
        # choose from:
        #    shibboleth, locallogin, google-oauth2, google-plus, yahoo, amazon,
        #    docker, dropbox-oauth2, facebook, launchpad, linkedin-oauth2,
        #    meetup, twitter

        # Add any additional settings by prefixing them with ADMINTOOL_EXTRA_SETTINGS_
        # Example:
        #ADMINTOOL_EXTRA_SETTINGS_SOCIAL_AUTH_TWITTER_KEY=93randomClientId
        #ADMINTOOL_EXTRA_SETTINGS_SOCIAL_AUTH_TWITTER_SECRET=ev3nM0r3S3cr3tK3y
        =======
        ADMINTOOL_DEBUG=True
        >>>>>>> Stashed changes

* As part of resolving the conflict, remove also the demarcation lines.  In this example, the ADMINTOOL_DEBUG setting was modified (uncommented) in the local file, while right on the following line, new settings were added in the upstream.  The correct resolution of thi confict is:

        ADMINTOOL_DEBUG=True
        ADMINTOOL_LOGIN_METHODS=google-oauth2 launchpad yahoo
        # choose from:
        #    shibboleth, locallogin, google-oauth2, google-plus, yahoo, amazon,
        #    docker, dropbox-oauth2, facebook, launchpad, linkedin-oauth2,
        #    meetup, twitter

        # Add any additional settings by prefixing them with ADMINTOOL_EXTRA_SETTINGS_
        # Example:
        #ADMINTOOL_EXTRA_SETTINGS_SOCIAL_AUTH_TWITTER_KEY=93randomClientId
        #ADMINTOOL_EXTRA_SETTINGS_SOCIAL_AUTH_TWITTER_SECRET=ev3nM0r3S3cr3tK3y

* After editing the file, indicate to Git the conflict has been resolved: `git reset`
* And dropped the stashed copy of the local modifications (git kept it because it ran into the conflict): `git stash drop`

## Updating docker containers

After updating the files driving the tools:

* Go into the respective directory (repeat for all of the three tools the updates apply to): `cd admintool`
* Pull the updated container images: `docker-compose pull`
* Restart the containers from updated images: `docker-compose up -d`
* Optionally, watch the logs (leave with Ctrl-C): `docker-compose logs`

# Appendix: Google Login

To get the Google credential (key+secret) to use in the admintool, do the following in the Google Developer Console:

* Start at http://console.developers.google.com/
* Create a new project
* From the main menu, select the API Manager
* In the list of available Google APIs, search for ````Google+ API```` and Enable this for your project.
* In the top-level API manager menu, select Credentials
* Configure the OAuth consent screen with how the application should be described to the user (at least, set Product name)
* Create a new Credential as an OAuth Client ID for a web application
* Add the Authorized redirect URI for your application - the form is (substitute your real hostname here):

        https://admin.example.org/accounts/complete/google-oauth2/

* After saving, this gives you the Client ID and secret (use these as the GOOGLE_KEY and GOOGLE_SECRET)

