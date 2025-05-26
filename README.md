# Install Redmine 6.0 on Ubuntu 24.04 LTS
steps to install Redmine 6.0.1 on Ubuntu Server 24.04.1 LTS with a minimal installation.

### What is Redmine:
Redmine is a versatile, open-source project management tool built on Ruby on Rails. It offers features like multi-project support, issue tracking, time tracking, and custom fields. Visit the official website at www.redmine.org to access a wealth of comprehensive information.

### Environment Created by This Procedure
* The environment created by this procedure is as follows:

Software	Version
Redmine	Redmine 6.0.1
OS	Ubuntu Server 24.04.1 LTS
Database	PostgreSQL 16.4
Web Server	Apache (Passenger is used for running Rails)
Ruby	3.3.6

## Installation of Necessary Packages

### Update the package management system database
```shell
sudo apt update
```

### Install development tools and header files necessary for building Ruby and Passenger
```shell
sudo apt install -y build-essential zlib1g-dev libssl-dev libreadline-dev libyaml-dev libcurl4-openssl-dev libffi-dev
```
### Install PostgreSQL and header files
```shell
sudo apt install -y postgresql libpq-dev
```
### Install Apache and header files
```shell
sudo apt install -y apache2 apache2-dev
```
### Install other tools
```shell
sudo apt install -y subversion git
```

## Installation of Ruby
### Download Source Code
Download the latest version of the Ruby 3.3 source code from the official Ruby website download page.
* https://www.ruby-lang.org/ja/downloads/
```shell
curl -O https://cache.ruby-lang.org/pub/ruby/3.3/ruby-3.3.6.tar.gz
```
* Redmine 6.0 supports Ruby versions 3.1 to 3.3. However, Ruby 3.1 will soon reach its End of Life (EOL), so please use Ruby 3.2 or later.
### Build Ruby
* Extract the downloaded Ruby tarball and build and install Ruby.
```shell
tar xvf ruby-3.3.6.tar.gz
cd ruby-3.3.6
./configure --disable-install-doc
make
sudo make install
cd ..
```
* If you specify --disable-install-doc when executing configure, you can avoid installing Ruby documentation and shorten the operation time.
- After the operation, run ruby -v to display the Ruby version and confirm that Ruby has been installed.
```shell
ruby -v
ruby 3.3.6 (2024-11-05 revision 75015d4c1f) [x86_64-linux]
```
## PostgreSQL Configuration
### Creating a User for Redmine
```shell
sudo -i -u postgres createuser -P redmine
Enter password for new role: (Enter a password for the PostgreSQL user redmine)
Enter it again: (Re-enter the password)
```
* Please set an arbitrary password for the PostgreSQL user redmine. This password will be used in the database.yml settings later.
### Creating a Database for Redmine
```shell
sudo -i -u postgres createdb -E UTF-8 -O redmine -T template0 redmine
```
## Redmine Installation
### Download Redmine
Use the svn command to download the latest source code of Redmine 6.0.x to the deployment directory of Redmine. In the following example, the source code is downloaded to the directory redmine under /var/lib/.
```shell
sudo mkdir /var/lib/redmine
sudo chown www-data /var/lib/redmine
sudo -u www-data svn co https://svn.redmine.org/redmine/branches/6.0-stable /var/lib/redmine
```
* Instead of using the svn command, you can also download the tarball from the following URL. But using the svn command is lower effort and easier.
http://www.redmine.org/projects/redmine/wiki/Download

## Database Connection Settings
* Create a file for settings connecting to the database from Redmine.
* Create config/database.yml under the Redmine installation directory (e.g., /var/lib/redmine) with the following content:
```shell
production:
  adapter: postgresql
  database: redmine
  host: localhost
  username: redmine
  password: "********"
  encoding: utf8
  ```
* ※ ******** part is the password for the redmine user created on PostgreSQL.
* ※ A configuration example is provided in config/database.yml.example for reference. However, the primary configuration examples in config/database.yml.example are for MySQL. Please note that you cannot use it as-is without modification for PostgreSQL.
### Create config/configuration.yml
Create a file that includes the settings connecting to the mail server.
Create config/configuration.yml under the Redmine installation directory with the following content:
```shell
production:
  email_delivery:
    delivery_method: :smtp
    smtp_settings:
      address: "localhost"
      port: 25
      domain: "example.com"
```
* Replace the part of example.com with the FQDN of the server running Redmine.
* A configuration example is provided in config/configuration.yml.example for reference.
* To enable mail sending, setup of MTA is required.
#### In configuration.yml, you can also set the storage location for uploaded files and database encryption.
### Move to Redmine Installation Directory
```shell
cd /var/lib/redmine
```
### Install gem Packages
Use Ruby's package management tool bundler to install the gem packages that Redmine depends on. Execute the following commands in the Redmine installation directory.
```shell
sudo bundle config set --local without 'development test'
sudo bundle install
```
## Initial Setup for Redmine
Perform initial setup related to the operation of Redmine. Execute the following commands in the Redmine installation directory.

### Create a Secret Token for Session Tampering Prevention
```shell
sudo -u www-data bin/rake generate_secret_token
```

### Create Database Tables
Create tables in the database specified in config/database.yml.
```shell
sudo -u www-data bin/rake db:migrate RAILS_ENV="production"
```

### Install Passenger
Install Phusion Passenger, used to run Rails applications like Redmine on Apache.
```shell
sudo gem install passenger -N
```

* -N is an option to omit the installation of documentation.

### Install Passenger Module for Apache
Execute the following command to build and install the module for Apache.
```shell
sudo passenger-install-apache2-module --auto --languages ruby
```

* Confirming Configuration Details for Apache
Executing the command below will display the configuration that should be added to Apache. This information will be used later when configuring Apache.
```shell
passenger-install-apache2-module --snippet
```

## Configuring Apache
Please create /etc/apache2/conf-available/redmine.conf with the following content:
```shell
# Configuration to allow access to Redmine's image files, CSS files, etc.
# By default in Apache 2.4, access to all files on the server is forbidden.

<Directory "/var/lib/redmine/public">
  Require all granted
</Directory>

# Basic configuration for Passenger.
# Describe the configuration displayed by passenger-install-apache2-module --snippet.
# The settings below will vary depending on the environment, so do not simply copy these lines as is, but always use the ones displayed by `passenger-install-apache2-module --snippet`.
#

LoadModule passenger_module /usr/local/lib/ruby/gems/3.3.0/gems/passenger-6.0.23/buildout/apache2/mod_passenger.so
<IfModule mod_passenger.c>
  PassengerRoot /usr/local/lib/ruby/gems/3.3.0/gems/passenger-6.0.23
  PassengerDefaultRuby /usr/local/bin/ruby
</IfModule>

# Add settings for Passenger tuning as necessary (optional).
# For more details, refer to Configuration reference - Passenger + Apache (https://www.phusionpassenger.com/docs/references/config_reference/apache/).
PassengerMaxPoolSize 20
PassengerMaxInstancesPerApp 4
PassengerPoolIdleTime 864000
PassengerStatThrottleRate 10

# Allow access to Redmine's installation directory
<Directory /var/lib/redmine/public>
    Allow from all
    Options -MultiViews
    Require all granted
</Directory>
```
### Applying Apache Configuration
Execute the following commands to ensure /etc/apache2/conf-available/redmine.conf is loaded at Apache startup and to apply the configuration to the running Apache.
```shell
sudo a2enconf redmine
apache2ctl configtest
sudo systemctl reload apache2
```
## Configure to Run Redmine on Apache with Passenger
The configuration depends on what URL format to run Redmine. Here are two examples.

### Pattern 1: Using the web server exclusively for Redmine
* This is the configuration for running Redmine in the root directory of the web server. Redmine can be accessed at http://server IP address or hostname/.

* Open /etc/apache2/sites-enabled/000-default.conf in an editor and change DocumentRoot to Redmine's public directory (e.g., /var/lib/redmine/public).
```shell
DocumentRoot /var/www/html
↓
DocumentRoot /var/lib/redmine/public
```

After changing the settings, restart Apache.
```shell
apache2ctl configtest
sudo systemctl reload apache2
```

### Pattern 2: Running Redmine in a Subdirectory
* Configure to access Redmine in a subdirectory of the URL (e.g., http://server IP address or hostname/redmine). This setting is convenient when running applications other than Redmine on the same server or when running multiple Redmines.

* Add the following configuration to the previously created Redmine-related Apache configuration file /etc/apache2/conf-available/redmine.conf. In the example below, the highlighted part represents the subdirectory name.
```shell
Alias /redmine /var/lib/redmine/public
<Location /redmine>
  PassengerBaseURI /redmine
  PassengerAppRoot /var/lib/redmine
</Location>
```
* Compile the assets such as stylesheets, javascripts and images. The value of RAILS_RELATIVE_URL_ROOT in the following command is the subdirectory specified for use in this example. When executing the command, specify the subdirectory name you have configured yourself.

```shell
sudo -u www-data bundle exec rake assets:precompile RAILS_ENV="production" RAILS_RELATIVE_URL_ROOT=/redmine
```
Execute the following commands to restart Apache.
```shell
apache2ctl configtest
sudo systemctl reload apache2
```
* If you encounter issues such as images not displaying or layout problems when accessing Redmine, execute the following command to remove the assets, and recompile them.
```shell
sudo -u www-data bundle exec rake assets:clobber RAILS_ENV="production"
```
