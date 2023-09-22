---

---
# EBM Installation Guide 
The following steps will convert a Mac Mini into a working EBM machine, ready to be hooked up to the printer hardware. 

It is very important to perform these steps in order, as many steps rely on setup from previous steps. 
Be sure to check the [troubleshooting doc](troubleshooting.md) if you run into an error.

Code snippets should be run in Terminal with administrative privileges. 

All of the following instructions, as well as machine operations, need to be run as a user with full Administrator privileges named 'ebm'. If this user account does not exist, create it and sign in.


## 1. Download and Install Dependencies

### a) Install XCode

Xcode can be found in the App Store. Install with default settings.


### b) Download this repository

The repository should be created in root for user *ebm* (`/Users/ebm` or `~`). 
Run:
```
cd ~
git clone https://[USERNAME]@bitbucket.org/teamodb/ebm-code.git
```
where [USERNAME] is your bitbucket username. You will be prompted for your bitbucket password. 

If the clone was successful, you will now see a new directory named 'ebm-code' in your user root.


### c) Download/Install MySQL Community Server 8. 

Download the DMG archive from the [MySQL website](https://dev.mysql.com/downloads/mysql/), then run the DMG file. The defaults settings are fine (install for all users, default install location). This will install under `/usr/local/mysql`.

>Remember what password you choose for root as you will need it later in the database setup step.


### d) Set up Python packages
Python 2.7 should, by default, come with Mac OSX. However, some modules need to be installed. 
```
sudo easy_install pip
sudo pip install MySQL-python
sudo pip install paramiko
sudo pip install cherrypy --upgrade --ignore-installed six
```


### e) Download/Install Qt.

Download [Open Source Qt](https://www.qt.io/download), using the Online Installer.
When asked what to install, choose *MacOS and Sources*. Install under `Users/ebm/Qt`. 

You may get an error "XCode is not installed" even though you have installed XCode. This is a bug on Apple's end and can be safely ignored.


### f) Install Homebrew
Homebrew is a tool which makes package installation easier. It will be used later in the setup process.
Run:
```
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

### g) Install printer drivers
Install the required drivers for the cover and block printers. Plugging the machines in should automatically install them. 
>**Warning**: This step *must* be performed before running the PAC Communicator. 

## 2. Configure MySQL Database

### a) Add MySQL to PATH
Add MySQL as an environment variable to the `PATH`. This can be done with the following command:
```
echo 'export PATH="/usr/local/mysql/bin:$PATH"' >> ~/.bash_profile
```

### b) Create database
Pull up the MySQL monitor terminal via the command line by running:
```
mysql -u root -p
```
and enter root password when prompted. Then run the following commands to set up the database and default user:
```
CREATE DATABASE IF NOT EXISTS odb;
CREATE USER 'odb'@'localhost' IDENTIFIED WITH mysql_native_password BY '[PASSWORD]';
GRANT ALL ON odb.* TO  'odb'@'localhost';
```
where [PASSWORD] is the password for the root user. 

### c) Create tables
Included in this repository under `setup/command_line` is a script named `setup_mysql_tables.sh`, which will run each file in the sql directory in order. Call it via
```
sh ~/ebm-code/setup/command_line/setup_mysql_tables.sh
```
and enter the MySQL root password when prompted. 

Alternatively, the tables can be created individually via running each script in the sql directory in the MySQL monitor terminal:
```
source /Users/ebm/ebm-code/sql/[TABLENAME].sql
```
where [TABLENAME] is the name of the table, for example available_block_printers.

**WARNING:** running these scripts will drop any existing tables. Be sure to back up the database if it has already been used.


## 3. PAC Configuration

### a) Add QT to PATH

Add QT as an environment variable to the   `PATH`:

```
echo 'export PATH="[PATH/TO/QT]:$PATH"' >> ~/.bash_profile
```


where [PATH/TO/QT] points to the bin directory for Qt, for example 
`/Users/ebm/Qt/5.12.3/clang_64/bin`


### b) Set up the QMySQL Driver


#### i) Compile QMYSQL to create dylib files
Some versions of Qt do not come with the QMYSQL Driver available. First, recompile the source via the 'configure' script.
```
cd ~/Qt/[VERSION]/Src
./configure -sql-mysql
make
```
where [VERSION] is the version of Qt, ex 5.12.3
> **NOTE:** This step takes several hours. This is a good time to go to lunch or take a nap. *Do not continue with installation until the compiler is finished.*

Once the build completes, go into the source for MySQL and make it:
```
cd /Users/ebm/Qt/[VERSION]/Src/qtbase/src/plugins/sqldrivers/mysql
qmake
make install
qmake -- MYSQL_PREFIX=/usr/local
make sub-mysql
```
where [VERSION] is the version of Qt, ex `5.12.3`

Once this is done, check to see if it was successful by going to the directory `/Users/ebm/Qt/[VERSION]/clang_64/plugins/sqldrivers`
and looking for libqsqlmysql.dylib.

>**NOTE:** This is a different directory from the one in the command line code. There are two directories named `sqldrivers`, one under `Src/qtbase/src` and one under `clang_64`. 


#### ii) Load QMYSQL driver

Qt needs to be told where to find MySQL. If it cannot find the MySQL Client Library, setup scripts and the PAC Communicator will fail with the error "QMYSQL driver not loaded."

First step is to check what version(s) of the MySQL Client Library you have. This is a Dynamic Library (dylib) file named "libmysqlclient" that lives within mysql's lib directory.

Run:
```
cd /usr/local/mysql/lib
ls -a
```

You will get a list of all of the files in that directory. Check if you have a file named exactly:
`libmysqlclient.dylib`

If you do not have this file, there should be a file 
`libmysqlclient.[X].dylib`

 where *[X]* is a version number. If this is the case, take note of the number.

If neither are present, you may need to install Connector/C. This can be done using Homebrew. Run:
```
brew install mysql-connector-c
```
Then check again for the MySQL client library in `/usr/local/mysql/lib`.

Once you have found the `libmysqlclient.dylib` file, check to see what library Qt is expecting. Go to the SQL Drivers directory under Plugins for the compiler (clang) you are using. In the Terminal, run:
```
cd ~/Qt/[VERSION]/clang_64/plugins/sqldrivers
otool -L libqsqlmysql.dylib
```
where *[VERSION]* is the version of Qt you installed (ex 5.12.3). (Make sure to type the filename correctly: it is `Lib Q SQL MySQL`)

This command will list the libraries that Qt's SQL Driver is looking for. The important one is `libmysqlclient`. Make sure that `libmysqlclient` points to the client file. 

If the client version is wrong, then while within the same directory, run
```
install_name_tool -change [OLD PATH] [NEW PATH] libqsqlmysql.dylib
```
Where *[OLD PATH]* is the directory currently listed by otool, and *[NEW PATH]* is the actual location of your client file. For an example:
```
install_name_tool -change @rpath/libmysqlclient.20.dylib /usr/local/mysql/lib/libmysqlclient.dylib libqsqlmysql.dylib
```


### c) Run *Printer Settings* 

Run the *Printer Settings* app inside the printer_settings directory, and choose the settings which match your current setup. If you would rather do this via terminal, you will need to manually insert a new row in the settings table via MySQL.
Note that the only printer drivers listed by this app are ones which already exist on the machine. If drivers are added later, re-run the app. 

### d) Compile the PAC Communicator
cd into PAC_Communicator and run
```
qmake
make
```
This will create the PAC Communicator app under /Users/ebm. 
>**Note**: The PAC Comm will crash on opening if you have not saved settings via the printer settings app (see previous step).

You can test the PAC Communicator in Island Mode (disconnected from PAC) via the following command:
```
./PAC_Communicator.app/Contents/MacOS/PAC_Communicator --island
```


If everything is set up properly, the PAC Communicator should run without crashing or producing errors. If not run in Island Mode it will likely hang but should not crash.


## 4. Webserver Configuration

### a) Set up Apache Server 
Apache should be included with the OS; all you need to do is to change the configuration and start it.

Replace the file in
```
/etc/apache2/httpd.conf
```
with the httpd.conf file in this repository.

To start the server, run 
```
sudo apachectl start
```
then test by opening Safari and going to "localhost" (nothing else, no http or .com, make sure it doesn't do a google search). 

### b) PHP Setup
Copy the included PHP.ini file to
```
/etc/php.ini
```

## 5. Network/Bundle Processor Setup

For security reasons, the credentials for the network connection are not included in this repository. In order to connect to the network to pull the network catalog, you will need to add these in manually to the table "credentials". You will additionally need to provide credentials to the order_tosser script. Please confirm each of these values before setup. 

### a) In the database
```
mysql odb -uodb -p
INSERT INTO credentials VALUES id=1, type='active', ready_url='https://web.odbnetwork.com/readyJobs', update_url='https://web.odbnetwork.com/update', ebm_id='[EBM ID]', ebm_passwd='[EBM PASSWORD]', sftp_hostname='island.odbnetwork.com', sftp_port='22', sftp_uname='[SFTP USERNAME]', sftp_passwd='[SFTP PASSWORD]', error_update_url='https://web.odbnetwork.com/errorUpdate', local_job_updater_url='https://web.odbnetwork.com/localEbmJobs';
```
where:

- [EBM ID] is the username for this particular EBM setup

- [EBM PASSWORD] is the password for the EBM ID

- [SFTP USERNAME] is the username for SFTP (separate from EBM ID)

- [SFTP PASSWORD] is the password for SFTP (separate from EBM password)


### b) In order_tosser
Create the file /bin/order_tosser.ini with the following:
```
net_uname   = [EBM ID]
net_passwd  = [EBM PASSWORD]
net_authkey = [NETWORK AUTHORIZATION KEY]
net_ip      = [STORE IP ADDRESS]
```

## 6. Set MySQLdb library paths
like with Qt, Python needs to know where the MySQL library is, along with some other dependencies (SSL and Crypto)
```
cd /Library/Python/[VERSION]/site-packages/MySQLdb/
otool -l _mysql.so
```
This will tell you where it thinks libmysqlclient is and what versions of libssl and libcrypto it's looking for.
```
install_name_tool -change @rpath/libmysqlclient.[version].dylib /usr/local/mysql/lib/libmysqlclient.dylib _mysql.so
install_name_tool -change libssl.[version].dylib /usr/local/opt/openssl/lib/libssl.[version].dylib _mysql.so
install_name_tool -change libcrypto.[version].dylib /usr/local/opt/openssl/lib/libcrypto.[version].dylib _mysql.so
```
where [version] matches what displayed from otool. You can also find the version by going to /usr/local/opt/openssl/lib and checking the number on the dylib file.

## 7. Set up GPG

### a) Install the GNUpg library
Run: 

```
sudo brew install gpgme
```

### b) Set up keys
In order to zip/unzip data bundles from the network you need the GPG keys and passphrase. The keys look something like this:

```
-----BEGIN PGP [PUBLIC/PRIVATE] KEY BLOCK-----
[hash]
-----END PGP [PUBLIC/PRIVATE] KEY BLOCK-----
```

You should have one private and one public, and additionally, have a short passphrase (approx 15 bytes). 
Create a file 

```
/opt/odb/gpg/passphrase.txt 
```

and paste the passphrase (no leading or trailing characters) inside.
Additionally, save the keys in txt files (location does not matter)
For both the public and private key, run in terminal:

```
gpg --import [PATH/TO/KEY.txt]
```

where [PATH/TO/KEY.txt] is the full path to the key file. For the secret key you will be prompted for the passphrase.

Now set the key to trusted. 

```
gpg --list-keys
gpg --edit-key [KEY] trust quit
```

where [KEY] is the key returned from list-keys. In this prompt, re-enter the passphrase, then choose option '5', then 'y' to confirm. 

### c) Connect to TTY
Now you need to set the GPG environment variable to use TTY correctly. Add this to PATH:

```
echo 'export GPG_TTY=$(tty)' >> ~/.bash_profile
```

### d) Allow automatic passphrase entry
Finally, configure GPG-Agent to allow for automatic passphrase entry. In the file

```
/Users/ebm/.gnupg/gpg-agent.conf
```

add the line

```
allow-loopback-pinentry
```

Note that gpg-agent.conf may not exist; if it does not, create it.

## 8. Makebook/Netmakebook

### a) Create necessary directories
The code is expecting a few extra directories which need full read, write, and execute permissions.
Run:
```
mkdir /opt/odb/catalog
mkdir /opt/odb/makebook
mkdir /opt/odb/netmakebook
mkdir /opt/odb/network

sudo chmod -R 777 /opt/odb/catalog
sudo chmod -R 777 /opt/odb/makebook
sudo chmod -R 777 /opt/odb/netmakebook
sudo chmod -R 777 /opt/odb/network
```

### b) Install dependencies
Makebook and netmakebook rely on three libraries:
```
brew install poppler
brew install ghostscript
brew install imagemagick
```

### c) Copy test book files into catalog

Move the five directories under setup/catalog into /opt/odb/catalog. 

### d) Create cron jobs
run
```
crontab -e
```
then paste the contents of setup/crontab.txt into the editor. 
To exit the editor, press the Esc key, then, :, then w, then q, then enter to save changes.



## 9. PAC Connection

The PAC lives on
```
http://192.168.0.250/views/main_a.html
```
and must be opened in a browser window. It also requires a specific extension to allow its legacy Java applet to run. The extension has been included under `setup/chrome_extension`.

In chrome, go to
chrome://extensions/

and turn on Developer Mode (upper right-hand corner). Then click "Load Unpacked" (upper left) and select the folder. 
When opening the PAC you will have to enable this extension.
