---
title: Install Kong Enterprise on RHEL
---

## Introduction

This guide walks through downloading, installing, and starting **Kong Enterprise** on **RHEL**.

The configuration shown in this guide is intended as an example. Depending on your
environment, you may need to make modifications and take measures to properly conclude
the installation and configuration.

Kong supports both PostgreSQL 9.5+ and Cassandra 3.11.* as its datastore. This guide provides
steps to configure PostgreSQL. For assistance in setting up Cassandra, please contact your Sales or Support representative.

## Prerequisites

To complete this installation you will need:

* A valid Bintray account. You will need your **username**, account **password** and account **API Key**.
  * Example:
    * **Bintray Access key**: `john-company`
    * **Bintray username**: `john-company@kong`
    * **Bintray password**: `12345678`
    * **Bintray API key**: `12234e314356291a2b11058591bba195830`
  * The API Key can be obtained by visiting [https://bintray.com/profile/edit](https://bintray.com/profile/edit) and selecting **API Key**
* A supported RHEL system with root equivalent access.
* A valid Kong Enterprise license JSON file, this can be found in your Bintray account. See [Accessing Your License](/enterprise/latest/deployment/access-license)

## Step 1. Prepare to Install Kong Enterprise and Download the License File

There are two options to install Kong Enterprise on RHEL. Both require a login to Bintray.

Log in to [Bintray](http://bintray.com). Your Kong Sales or Support contact will assign credentials to you.

{% navtabs %}
{% navtab Download RPM file %}

1. Go to: https://bintray.com/kong/kong-enterprise-edition-rpm/rhel.
2. Select the latest Kong version from the list.
3. From the Kong version detail page, select the **Files** tab.
4. Select the RHEL version appropriate for your environment. e.g. `RHEL` -> `8`.
5. Save the RPM file available: e.g. `kong-enterprise-edition-1.3.0.1.rhel8.noarch.rpm`
6. Copy the RPM file to your home directory on the RHEL system. You may use a command like:

    ```bash
    $ scp kong-enterprise-edition-1.3.0.1.rhel8.noarch.rpm <rhel user>@<server>:~
    ```

*Optional:* The following steps are for verifying the integrity of the package. They are not necessary to move on to [installation](#option-1-if-installing-using-a-downloaded-rpm-package).

7. Kong's official Key ID is `2cac36c51d5f3726`. Verify it by querying the RPM package and comparing it to the Key ID:

    ```bash
    $ rpm -qpi kong-enterprise-edition-1.3.0.1.rhel8.noarch.rpm | grep Signature
    ```

8. Download Kong's official public key to ensure the integrity of the RPM package:

    ```bash
    $ curl -o kong.key https://bintray.com/user/downloadSubjectPublicKey?username=kong
    $ rpm --import kong.key
    $ rpm -K kong-enterprise-edition-1.3.0.1.rhel8.noarch.rpm
    ```

9. Verify you get an OK check. You should have an output similar to this:

    ```
    kong-enterprise-edition-1.3.0.1.rhel8.noarch.rpm: digests signatures OK
    ```  

{% endnavtab %}
{% navtab Download Kong repo file and add to Yum repo %}

1. Click this URL to download the Kong Enterprise RPM repo file: https://bintray.com/kong/kong-enterprise-edition-rpm/rpm.

2. Edit the repo file using your preferred editor and alter the baseurl line as follows:

    ```
    baseurl=https://USERNAME:API_KEY@kong.bintray.com/kong-enterprise-edition-rpm/rhel/RELEASEVER
    ```

    Replace `USERNAME` with your Bintray account user name.
    Replace `API_KEY` with your Bintray API key. You can find your key on your Bintray profile page at https://bintray.com/profile/edit and selecting the API Key menu item.
    Replace `RELEASEVER` with the major RHEL version number on your target system. For example, for version 7.7.1908, the appropriate `RELEASEVER` replacement is 7.

    The result should look something like this:
    ```
    baseurl=https://john-company:12234e314356291a2b11058591bba195830@kong.bintray.com/kong-enterprise-edition-rpm/rhel/8
    ```

3. Securely copy the changed repo file to your home directory on the RHEL system:

    ```bash
    $ scp bintray--kong-kong-enterprise-edition-rpm.repo <rhel user>@<server>:~
    ```

{% endnavtab %}
{% endnavtabs %}

### Download your Kong Enterprise License

1. Download your license file from your account files in Bintray: `https://bintray.com/kong/<YOUR_REPO_NAME>/license#files`

2. Securely copy the license file to your home directory on the RHEL system:

    ```
    $ scp license.json <rhel username>@<server>:~
    ```

### Result

You should now have two files in your home directory on the target RHEL system:
- Either the Kong RPM or Kong Yum repo file.
- The license file `license.json`

## Step 2. Install Kong Enterprise

{% navtabs %}
{% navtab Using downloaded RPM package %}

1. Install EPEL (Extra Packages for Enterprise Linux), if not already installed:

    ```bash
    $ sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm -y
    $ sudo yum install epel-release -y
    ```

2. Execute a command similar to the following, using the appropriate RPM file name you downloaded:

    ```bash
    $ sudo yum install kong-enterprise-edition-1.3.0.1.rhel8.noarch.rpm -y
    ```

{% endnavtab %}
{% navtab Using Yum repo %}

1. Move the repo file in your home directory to the /etc/yum.repos.d/ directory:

    ```bash
    $ sudo mv bintray--kong-kong-enterprise-edition-rpm.repo /etc/yum.repos.d/
    ```

2. Begin the installation using the Yum repository:

    ```bash
    $ sudo yum update -y
    $ sudo yum install kong-enterprise-edition -y
    ```    
{% endnavtab %}
{% endnavtabs %}


### Copy the License File

Copy the license file from your home directory to the `/etc/kong` directory:

```bash
$ sudo cp license.json /etc/kong/license.json
```

## Step 3. Setup PostgreSQL

1. Install PostgreSQL.

    Follow the instructions avaialble at https://www.postgresql.org/download/linux/redhat/ to install a supported version of PostgreSQL. Kong supports version 9.5 and higher. As an example, you can run a command set similar to:

    ```bash
    $ sudo dnf install postgresql-server
    ```

2. Initialize the PostgreSQL database and enable automatic start.

    ```bash
    $ sudo /usr/bin/postgresql-setup initdb
    $ sudo systemctl enable postgresql
    $ sudo systemctl start postgresql
    ```

3. Switch to PostgreSQL user and launch PostgreSQL.

    ```bash
    $ sudo -i -u postgres
    $ psql
    ```

4. Create a Kong database with a username and password.

    > ⚠️**Note**: Make sure the username and password for the Kong Database are
    > kept safe. This example uses a simple username and password for illustration purposes only. Note the database name, username and password for later.

    ```bash
    $ psql> CREATE USER kong; CREATE DATABASE kong OWNER kong; ALTER USER kong WITH password 'kong';
    ```

5. Exit from PostgreSQL and return to your terminal account.

    ```bash
    $ psql> \q
    $ exit
    ```

6. Edit the the PostgreSQL configuration file `/var/lib/pgsql/data/pg_hba.conf` using your preferred editor.

    Under IPv4 local connections replace `ident` with `md5`:

    | Protocol   	| Type 	| Database 	| User 	| Address      	| Method 	|
    |------------	|------	|----------	|------	|--------------	|--------	|
    | IPv4 local 	| host 	| all      	| all  	| 127.0.0.1/32 	| md5    	|
    | IPv6 local 	| host 	| all      	| all  	| 1/128        	| ident  	|

    PostgreSQL uses `ident` authentication by default. To allow the `kong` user to communicate with the database locally, change the authentication method to `md5` by modifying the PostgreSQL configuration file.

7. Restart PostgreSQL.

    ```bash
    $ sudo systemctl restart postgresql
    ```

## Step 4. Modify Kong's configuration file to work with PostgreSQL

1. Make a copy of Kong's default configuration file.

    ```bash
    $ sudo cp /etc/kong/kong.conf.default /etc/kong/kong.conf
    ```

2. Uncomment and update the PostgreSQL database properties in `/etc/kong/kong.conf` using your preferred text editor.

    ```
    pg_user = kong
    pg_password = kong
    pg_database = kong
    ```


## Step 5. Seed the Super Admin password and bootstrap Kong

Setting a password for the **Super Admin** before initial start-up is strongly recommended. This will permit the use of RBAC (Role Based Access Control) at a later time, if needed.

1. Create an environment variable with the desired **Super Admin** password and store the password in a safe place. Run migrations to prepare the Kong database:

    ```bash
    $ sudo KONG_PASSWORD=<password-only-you-know> /usr/local/bin/kong migrations bootstrap -c /etc/kong/kong.conf
    ```

3. Start Kong Enterprise:

    ```bash
    $ sudo /usr/local/bin/kong start -c /etc/kong/kong.conf
    ```

4. Verify Kong Enterprise is working:

    ```bash
    $ curl -i -X GET --url http://localhost:8001/services
    ```

5. You should receive a `HTTP/1.1 200 OK` message.

## Step 6. Finalize Configuration and Verify Installation

### Enable and Configure Kong Manager

To access Kong Enterprise's Graphical User Interface, Kong Manager, update the `admin_gui_url` property in `/etc/kong/kong.conf` file the to the DNS, or IP address, of the RHEL system. For example:

    ```
    admin_gui_url = http://<DNSorIP>:8002
    ```

This setting needs to resolve to a network path that will reach the RHEL host.

1. It is necessary to update the administration API setting to listen on the needed network interfaces on the RHEL host. A setting of `0.0.0.0:8001` will listen on port `8001` on all available network interfaces.

    ```
    admin_listen = 0.0.0.0:8001, 0.0.0.0:8444 ssl
    ```

2. You may also list network interfaces separately as in this example:

    ```
    admin_listen = 0.0.0.0:8001, 0.0.0.0:8444 ssl, 127.0.0.1:8001, 127.0.0.1:8444 ssl
    ```

3. Restart Kong for the setting to take effect:

    ```bash
    $ sudo /usr/local/bin/kong restart
    ```

4. You may now access Kong Manager on port `8002`.

### Enable the Developer Portal

Kong Enterprise's Developer Portal can be enabled by setting the `portal` property to `on` and setting the `portal_gui_host` property to the DNS, or IP address, of the RHEL system. For example:

```
portal = on
portal_gui_host = <DNSorIP>:8003
```

1. Restart Kong for the setting to take effect:

    ```bash
    $ sudo /usr/local/bin/kong restart
    ```

2. The final step is to enable the Developer Portal. To do this, execute the following command, updating `DNSorIP` to reflect the IP or valid DNS for the RHEL system.

    ```bash
    $ curl -X PATCH http://<DNSorIP>:8001/workspaces/default   --data "config.portal=true"
    ```

3. You can now access the Developer Portal on the default workspace with a URL like:

    ```
    http://<DNSorIP>:8003/default
    ```

## Troubleshooting

If you did not receive an `HTTP/1.1 200 OK` message, or need assistance completing
your setup, reach out to your Kong Support contact or go to the
[Support Portal](https://support.konghq.com/support/s/).

## Next Steps

Check out Kong Enterprise's series of
[Getting Started](/enterprise/latest/getting-started) guides to get the most
out of Kong Enterprise.
