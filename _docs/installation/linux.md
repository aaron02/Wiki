---
title: Linux Guide
type: installguide
category: 2
layout: single_markdown
---

### Linux Guide

Please note: this guide has been written with the objective of setting up a Linux server running a generic kernel/OS, like Ubuntu/Debian.
{: .success }
This guide file has been written with strict use of the console in mind so it suites both Linux desktop and server users. Linux was made to run command line, so there isn’t an easier, quicker way to do things than the way we are about to do them.
{: .info }

### Initial Setup

First, having presumably installed a fresh copy of Linux, we need to update our server so that we can compile AscEmu. This will require several different packages. 
For the following commands, log in as the Linux root administrator.

```console
sudo apt-get update && sudo apt-get dist-upgrade
sudo apt-get install g++ git-core git cmake build-essential zlib1g-dev libssl-dev libpcre3-dev libbz2-dev
```

### MySQL Setup

First we need to install MySQL into Linux as well as make sure that we have the correct libraries to properly operate it.

<pre>
Only for Debian.
echo -e "deb http://repo.mysql.com/apt/debian/ stretch mysql-5.7\ndeb-src http://repo.mysql.com/apt/debian/ stretch mysql-5.7" > /etc/apt/sources.list.d/mysql.list
wget -O /tmp/RPM-GPG-KEY-mysql https://repo.mysql.com/RPM-GPG-KEY-mysql
apt-key add /tmp/RPM-GPG-KEY-mysql
apt update
</pre>

```console
sudo apt-get install mysql-server mysql-client libmysqlclient-dev
```

By default Ubuntu 18.04 and later use auth_socket for MySQL root password. Let's set up a strong password for root by logging into MySQL.

```console
sudo mysql
```
Create the strong and secure password with the following query.

```console
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'your_password';
```
You can now quit the MySQL terminal by simply entering *quit* or *exit*.

To have access to your new MySQL server from other computers, open up the MySQL configuration file with your favourite text editor.

```console
sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf
```

And comment out the following line (by adding the #):

```console
#bind-address = 127.0.0.1
```

Then save and exit the file and restart the MySQL service.

```console
sudo /etc/init.d/mysql restart
```

That's it! Setting up MySQL is pretty straight forward.

### OpenSSL

Some Linux distributions, such as Ubuntu 22.04 LTS, come with incompatible version of OpenSSL.
To make sure your Linux has correct version run following command:

```console
openssl version
```

If the output says "OpenSSL 1.X.X" you can skip this section and continue on Security and Accounts.
However, if it says "OpenSSL 3.X.X" or higher, you must downgrade your OpenSSL to 1.X.X version. Client supports up to OpenSSL 1.1.1 only.

First, install a new package.

```console
sudo apt-get install checkinstall
```

We will install OpenSSL 1.1.1 alongside your existing OpenSSL version in /usr/local/. Let's change directory.

```console
cd /usr/local/src/
```

Next you must find latest 1.1.1 version from [OpenSSL website](https://www.openssl.org/source/). As of writing this the latest version is 1.1.1q.
Copy the download link and use wget to download it.

```console
sudo wget https://www.openssl.org/source/openssl-1.1.1q.tar.gz
```

Extract the content and navigate into it.

```console
sudo tar -xf openssl-1.1.1q.tar.gz
cd openssl-1.1.1q
```

Before compiling let's setup OpenSSL config.

```console
sudo ./config --prefix=/usr/local/ssl --openssldir=/usr/local/ssl shared zlib
```

Now it's ready for compile. Run the following commands:

```console
sudo make
sudo make test
sudo make install
```

OpenSSL is now ready for use, but we must still do some changes to tell your Linux to use this version of OpenSSL.
Create a new config file with your favourite text editor. Replace 1.1.1q with your downloaded version of OpenSSL.

```console
sudo nano /etc/ld.so.conf.d/openssl-1.1.1q.conf
```

Add following line to the file:

```console
/usr/local/ssl/lib
```

Save the file and close it.

Next reload the dynamic link.

```console
sudo ldconfig -v
```

Backup the existing OpenSSL binaries.

```console
sudo mv /usr/bin/c_rehash /usr/bin/c_rehash.backup
sudo mv /usr/bin/openssl /usr/bin/openssl.backup
```

Last but not least, add **:/usr/local/ssl/bin** to the end of your PATH.
Make sure you add it inside the quotation marks.

```console
sudo nano /etc/environment
```

Save the file and reload environment.

```console
source /etc/environment
```

Now you can test it by checking OpenSSL version again.
You should see "OpenSSL 1.1.1q" (or your downloaded version) if everything went right.

```console
openssl version
```

<sub>Credits for this section to Aviscall01@getmangos.eu</sub>

### Security and Accounts

Once that is complete, we have the right environment in Linux to compile the server.  Before we can compile though we need to address some very serious security issues.  Whatever distro you are using, whether your server is private or public. 
{: .info }
Please do NOT run your AscEmu server using your root account.
{: .error }

Having said that, lets move on to create a basic account in Linux from which you will run AscEmu. You can name this account anything you would like, but for the sake of standardization, we will name ours ascemu.  While still in your root account type:

```console
sudo useradd -m -s /bin/bash ascemu
```

Next you will create a password for the account with the following command.

```console
sudo passwd ascemu
```

Once you have created the ascemu user, you will have a new directory in **/home/ascemu/**. This will be the working root directory for your installation and eventually, for running the server.

In the commands below, ~ is used as a shorthand for the current user's home directory, which we assume to be /home/ascemu. If for whatever reason you cannot use ~, simply replace it with /home/ascemu.
{: .info }

### Getting the Files

First, switch to the AscEmu account, or whichever account you have just created.

```console
sudo su - ascemu
```

Next, we need to download the AscEmu source files to compile them.  Let's make sure we are in the home directory:

```console
cd ~
```

I am a fan of organization, so let's make some directories and organize this mess.  We will create an installer, server, and ascemu directory so that we can keep all of our files straight.  The installer directory may seem like a waste for now, but it will come into play later when we install the database.

```console
mkdir ~/installer
```

```console
mkdir ~/server
```

As you may have guessed, the installer directory will contain AscEmu source files, and the server directory will contain the actual compiled files (like the libraries and binaries) to run the server.  The next step is to download the source files, so we will change to our installer/ascemu_code directory and use git to get the files.

```console
mkdir ~/installer/ascemu_code
cd ~/installer/ascemu_code
```
With the -b required_branch, you can select a branch.
{: .info }

```console
git clone -b master git://github.com/AscEmu/AscEmu.git ~/installer/ascemu_code
```

Update the code to the current version.

```console
cd ~/installer/ascemu_code
git pull origin master
```

### Compiling

Once we have the source files we can start compiling AscEmu. The first step is to create a configuration file that will be used to pass the variables to the make file so that AscEmu will compile properly.

```console
mkdir ~/installer/build
```

```console
cd ~/installer/build
```

```console
cmake -DCMAKE_INSTALL_PREFIX=~/server -DCMAKE_BUILD_TYPE=Release -DBUILD_WITH_WARNINGS=0 -DBUILD_TOOLS=0 -DASCEMU_VERSION=WotLK ../ascemu_code
```

If you needed to downgrade your OpenSSL version earlier you must add some additional variables:

```console
cmake -DCMAKE_INSTALL_PREFIX=~/server -DCMAKE_BUILD_TYPE=Release -DBUILD_WITH_WARNINGS=0 -DBUILD_TOOLS=0
-DASCEMU_VERSION=WotLK -DOPENSSL_LIBRARIES=/usr/local/ssl/lib -DOPENSSL_INCLUDE_DIR=/usr/local/ssl/include ../ascemu_code
```

Here is a quick view of the variables you can include with cmake command:

<pre>
-DCMAKE_INSTALL_PREFIX = the location where AscEmu binaries are installed
-DCMAKE_BUILD_TYPE = choose from Release or Debug mode
-DBUILD_WITH_WARNINGS = 0 (disabled) or 1 (enabled)
-DBUILD_TOOLS = 0 (disabled) or 1 (enabled)
-DASCEMU_VERSION = choose from Classic, TBC, WotLK, Cata or MoP
-DUSE_PCH = 0 (disabled) or 1 (enabled) - Precompiled headers are enabled by default
</pre>

Then we now simply invoke make and make install to install to the prefix directory.

```console
make && make install
```

If you have a multicore machine, then you can substitute that final command with this one, where x is equal to the number of cores + 1.  For example, with 2 cores x would be 3.
If you want to use all your cores just use a large number like 15.
{: .info }

```console
make -j x && make install
```

This will not effect your server, this will only tell "make" to compile using all of your available CPU power.
{: .info }

If this last step is successful then you are ready to configure your server and get on your way.

### DBC and Map Files

Next you will transfer the DBC and map files over to your server.

Use Wine if you must but preferably a Windows machine to extract the DBC and map files.
{: .info }

```console
mkdir ~/server/dbc
```

```console
mkdir ~/server/maps
```

Place the DBC and map files in their respective directories above.

### Config Files

All that is left to do is to create the /configs/ directory and move the configuration files there, and make the AscEmu binaries executable.

```console
cd ~/server
mkdir configs
mv ~/installer/ascemu_code/configs/*.conf ~/server/configs
chmod a+x logon
chmod a+x world
```

Now your configuration files are in the .../configs folder ready to be edited, and used by the AscEmu server and your AscEmu binaries are executable.

### Database Setup

It's best to not run AscEmu with your MySQL root account. In this step you will create a new MySQL user which will only have access to your AscEmu MySQL databases.

First, login to MySQL terminal with your MySQL root account.

```console
mysql -u root -p
```

Next you will create the MySQL user. Please change the respective usernames and passwords to your own unique variants!

```console
CREATE USER 'your_username'@'%' IDENTIFIED BY 'your_password'
WITH MAX_QUERIES_PER_HOUR 0 MAX_CONNECTIONS_PER_HOUR 0 MAX_UPDATES_PER_HOUR 0 MAX_USER_CONNECTIONS 0;
```

Create the databases. Replace *your_username* with the username you used in the previous step.

```console
CREATE DATABASE `ascemu_world`;
GRANT ALL PRIVILEGES ON ascemu_world.* TO 'your_username'@'%';
CREATE DATABASE `ascemu_char`;
GRANT ALL PRIVILEGES ON ascemu_char.* TO 'your_username'@'%';
CREATE DATABASE `ascemu_logon`;
GRANT ALL PRIVILEGES ON ascemu_logon.* TO 'your_username'@'%';
quit
```

After you have created the databases, it's time to populate them with the SQL files.

### World Database

AscEmu supports only OneDB world database which is maintained by AscEmu team. OneDB, as the name suggests, has all versions (Classic - Mop) packed into a single database. ![github.svg](/Wiki/images/mark-github.svg) [Link to Github](https://github.com/AscEmu/OneDB/).
{: .info }

First, switch back to your installer folder and use git to clone the OneDB repository.

```console
cd ~/installer
git clone git://github.com/AscEmu/OneDB.git onedb
```

Go into 'fullDB' folder.

```console
cd ~/installer/onedb/fullDB
```

Next run this command to import all SQL files from the folder into the world database. Replace your_username with the username you created in the previous section.

This can take a while.

```console
cat *.sql | mysql -u your_username -p ascemu_world
```

Now your world database is populated.

### Logon and Character Databases

Let's move the full SQL folder from AscEmu code to the server folder.

```console
cd ~/server
cp -r ~/installer/ascemu_code/sql ./
cd ~/server/sql
```

Now, let's populate the databases. Again, replace your_username with the username you created in Database Setup.

```console
mysql -u your_username -p ascemu_logon < logon/logon_base.sql
mysql -u your_username -p ascemu_char < character/character_base.sql
```

The logonserver and worldserver will automatically execute new or missing SQL updates from the SQL folder.

Please note that when there are new SQL updates in the future, they will not automatically transfer to your server folder. You will need to copy the SQL folder again from installer/ascemu/code folder, or apply the updates manually into your database one by one.
{: .info }

### Configuration Files

Use an editor of your choice, in this example it'll be **nano**. Make sure to read all config files at least once, so you know what configuration is where and you don't end up with an administrator account with default password you didn't know about ;)

```console
cd ~/server/configs
nano logon.conf
nano world.conf
```

### Configuring logon.conf

Enter your MySQL information you created in Database Setup at the the following section.

```console
<LogonDatabase Hostname = "localhost"
Username = "ascemu"
Password = "ascemu"
Name     = "ascemu_logon"
Port     = "3306">
```

### Configuring world.conf

Enter your MySQL information you created in Database Setup at the the following section. 

```console
<WorldDatabase Hostname = "localhost" Username = "ascemu" Password = "ascemu" Name = "ascemu_world" Port = "3306">
<CharacterDatabase Hostname = "localhost" Username = "ascemu" Password = "ascemu" Name = "ascemu_char" Port = "3306">
```

That's it! You should now have a fully functioning copy of AscEmu. Look at the sections below for handy scripts and how to create an account.

### Using Screen

You may also wish to preserve the AscEmu processes once you've logged out of SSH. To do this, you'll need to install Screen if it isn't installed already.

The syntax is relatively simple.

```console
screen ./world
```

This will run the 'world' program in a new session. To leave this screen and keep it running, press "CTRL + A and then D" to detach.

To return to the screen later on use

```console
screen --list
```

then type

```console
screen -r ScreenIDHere
```

You may replace ./world with sh AE_Restarter.sh for example, to run a restart program or even use screen on startup to launch the server.

To do this, you can either use a restart script, cronjob, or if you are working with a desktop Linux environment you can simply use the built in startup application manager(Both Gnome and KDE have these).

If you wish to set up a cronjob, google around - far more documentation is provided for both cron AND Screen then the AscEmu team can provide on these programs.

### Sample restart script

```console
#! /bin/bash
while :
do
./world
sleep 150
wait $!
echo Realm went down, restarting!
done
```

Increase 150 to 250 or 300 to prevent bash from spamming the executable or the script may cause undesired effects if the server refuses to start or has an error.
{: .info }

Save the above script as AE_Restarter.sh and call it using

```console
screen ./AE_Restarter.sh
```

You are now ready to move on to create an account.

### Create an Account

**1.** Go to your world console and create an account.

```console
createaccount <accountname> <password>
```

example: createaccount admin adminpassword

**2.** Change permission(gmlevel) in your world console:

```console
setaccpermission <accountname> <permission>
```

example: setaccpermission admin az

**For further information about access levels, have a look to this page [GM Access Levels](/Wiki/docs/commands/access_levels/ "GM Access Levels")**
