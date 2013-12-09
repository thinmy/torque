## Torque for Android: PHP/MySQL Config ##

This repo contains a set of scripts and instructions to setup a minimally functional server/database for uploading ODB2 data logged from your car in real- time using the [Torque Pro](https://play.google.com/store/apps/details?id=org.prowl.torque) app for Android.

You can also find a growing set of tools to manage, analyze, and visualize the data uploaded into the database from your Torque app.


### Setup ###

These instructions assume you already have a LAMP-like server or have access to one. Specifically, you'll need the following:

  * MySQL database
  * Apache webserver
  * PHP server-side scripting

Everything was tested on a computer running Ubuntu 12.04 with MySQL 5.5, Apache 2.2, and PHP 5.3, but many other configurations will work.

If you need help setting up these prerequisites, there is obviously a ton of information on Google on how to configure a LAMP server, but I'd recommend one of [these guides](https://library.linode.com/lamp-guides/ubuntu-12.04-precise-pangolin) by Linode or [this guide](https://help.ubuntu.com/community/ApacheMySQLPHP) by Ubuntu.

You don't necessarily need to do everything exactly the same way as I do it here, but my intentions are to provide a reliable configuration that incorporates as many best practices as possible.


##### Create Empty MySQL Database & Configure User #####

First we'll create an empty database that will be configured further in the next section. Once we have an empty database, we'll then create a MySQL user and provide them with the necessary permissions on the database.

Start by opening up a MySQL shell as the root user:

```bash
mysql -u root -p
```

When prompted, enter the password for the MySQL root user. Once logged in, create a database named `torque`:

```sql
CREATE DATABASE torque;
```

Before doing anything else with the database, create a user with permission to insert and read data from the database. In this tutorial, we'll create a user `steve` with password `zissou44` that has access to all tables in the database `torque` on `localhost`:

```sql
CREATE USER 'steve'@'localhost' IDENTIFIED BY 'zissou44';
GRANT ALL PRIVILEGES ON torque.* TO 'steve'@'localhost';
FLUSH PRIVILEGES;
```

Exit the MySQL shell by typing `quit` and we'll move onto creating an [Options File](https://dev.mysql.com/doc/refman/5.5/en/option-files.html) for this new MySQL user. An options file is a MySQL configuration file will allow us to login to the database without typing out the username and password each time, but in a way that is just as secure.

Create a file in your home directory called `.my.cnf` (e.g. */home/myuser/.my.cnf*) and enter the following text into it, replacing the user/password with the one you made:

```
[client]
user="steve"
password="zissou44"
```

To protect the contents of this file, set the permissions on it so only the owner of the file (your system user) can read it and write to it:

```bash
chmod 600 ~/.my.cnf
```


##### Create MySQL Table #####

Next we'll create a table in the database to store the log data sent from Torque. Fortunately, I've provided a shell script in this repo that will do this for you. Open a terminal in the folder where you cloned this repo and,assuming you put your MySQL options file in your home directory, simply run:

```bash
mysql < create_torque_log_table.sql
```

##### Configure Webserver #####


At this point, the MySQL settings are all configured. The only thing left to do related to the database is to add your MySQL user/password to the PHP script. Open the `torque.php` file and enter your MySQL user and password in the blank **$db_user** and **$db_pass** fields:

```php
...
// Establish db connection
$db_host = "localhost";
$db_user = "steve";     // Enter your MySQL username
$db_pass = "zissou44";  // Enter your MySQL password
$db_name = "torque";
$db_table = "raw_logs";
...
```

Now move the `torque.php` file to your webserver and set the permissions. Assuming the document root for your Apache server is located at `/var/www`, you could do:

```bash
mkdir /var/www/torque
cp torque.php /var/www/torque/
chmod 755 /var/www/torque/
chmod 644 /var/www/torque/torque.php
```

The last two lines set the permissions seperately for the directory we made and the PHP file. In general, directories on your webserver should have 755 permissions and files should have 644. You can clean up the permissions across your entire server for directories and files by executing the following pipes:

```bash
find /var/www/ -type d -print0 | xargs -0 chmod 755
find /var/www/ -type f -print0 | xargs -0 chmod 644
```

With `torque.php` in place on your webserver, it's time to make sure everything works.


##### Testing #####

To use your webserver with Torque, the domain name mapped to the webserver needs to be available to the remote Internet.

Assuming your domain is `http://www.example.com`, open a browser and navigate to `http://www.example.com/torque/torque.php` and make sure your script returns the "OK!" message and only that. If that worked, you're ready to enter the URL to your PHP script into your Torque app.


##### Configure Torque Settings #####

To use your database/server with Torque, open up the Torque app on your phone and go to `Settings` -> `Data Logging & Upload` -> `Webserver URL`. Enter the URL to your **torque.php** script (e.g. `www.example.com/torque/torque.php`) and press `OK`.

With your URL inputted into Torque, test to make sure it works by clicking `Test settings`. If the message returns saying "No problems found!", then everything is in working order. Congratulations!

The final thing you'll want to do before going for a drive is to check the appropriate boxes on the `Data Logging & Upload` page under the `REALTIME WEB UPLOAD` section. Personally, I have both **Upload to webserver** and **Only when ODB connected** checked.

At this point, you should be all setup. The next time you connect to Torque in your car, data will begin syncing into your MySQL database in realtime!


### Getting Data From the Database ###


Once you've collected some data in the database, you will eventually want to get it out and look at it. In this repo you'll find a shell script `dbdump_to_csv.sh` which will dump all of the data out of the database into a nicely formatted CSV file. It uses the `~/.my.cnf` file created earlier to login to the database and creates a folder `torque_data` in the repo (the first time it is run) before putting creating a CSV file named with today's date in the folder.

The `dbdump_to_csv.sh` script would work well as a cronjob if you wanted to create a CSV file every day with your data. Otherwise to run it manually, simply do:

```bash
sh ./dbdump_to_csv.sh
```


### Coming Soon ###

  * Create dynamic visualizations of the data in the database inside a webapp.
  * Provide Python scripts that use [pandas](http://github.com/pydata/pandas) to parse/clean the dumped CSV files and perform analyses on the data.


