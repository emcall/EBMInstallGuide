---

---
# EBM Installation Guide 
The following steps will convert a Mac Mini into a working EBM machine, ready to be hooked up to the printer hardware or to serve as a work environment for engineers improving the code. 

It is very important to perform these steps in order, as many steps rely on setup from previous steps. 
Be sure to check the troubleshooting doc if you run into an error.

Code snippets should be run in Terminal with administrative privileges. 

All of the following instructions, as well as machine operations, need to be run as an Administrator.


## 1. Download and Install Dependencies


### a) Create user 'ebm'

If there is no user account named 'ebm', create it by going to System Settings, then Users and Groups. Give the account full Administrator privileges. Sign into this account.


### b) Install XCode

Xcode can be found in the App Store. Install it with default settings.


### c) Download this repository

The repository should be cloned into root for the user *ebm* (`/Users/ebm` or `~`). 
To clone the repository, run:
```
cd ~
git clone https://[USERNAME]@bitbucket.org/teamodb/ebm-code.git
```
where [USERNAME] is your bitbucket username. You will be prompted for your bitbucket password. 

If the clone was successful, you will now see a new directory named 'ebm-code' in your user root.


### d) Download/Install MySQL Community Server 8. 

Download the DMG archive from the [MySQL website](https://dev.mysql.com/downloads/mysql/), then run the DMG file. The defaults settings are fine (install for all users, default install location). This will install under `/usr/local/mysql`.

>Remember what password you choose for root as you will need it later in the database setup step.


### e) Install Python packages
Python 2.7 comes by default with Mac OSX. However, the codebase uses a few packages that are not installed by default. 
To install these packages, run: 
```
sudo easy_install pip
sudo pip install MySQL-python
sudo pip install paramiko
sudo pip install cherrypy --upgrade --ignore-installed six
```


### f) Download/Install Qt.

Download [Open Source Qt](https://www.qt.io/download), using the Online Installer.
When asked what to install, choose *MacOS and Sources*. Install under `Users/ebm/Qt` . 

You may get the error "XCode is not installed" even though you have installed XCode. This is a bug on Apple's end and can be safely ignored.


### g) Install Homebrew
Homebrew is a tool that makes package installation easier. It will be used later in the setup process.
Run:
```
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```


### h) Install printer drivers
Install the required drivers for the cover and block printers. Plugging the machines in should automatically install them. 

## 2. Configure MySQL Database

### a) Add MySQL to PATH
MySQL must be added an environment variable to the `PATH`. 
To add MySQL to `PATH`, run:
```
echo 'export PATH="/usr/local/mysql/bin:$PATH"' >> ~/.bash_profile
```

### b) Create database
Pull up the MySQL monitor terminal via the command line by running:
```
mysql -u root -p
```
Enter the root password from step 1d when prompted. Then, to set up the database and default user, run:
```
CREATE DATABASE IF NOT EXISTS odb;
CREATE USER 'odb'@'localhost' IDENTIFIED WITH mysql_native_password BY '[PASSWORD]';
GRANT ALL ON odb.* TO  'odb'@'localhost';
```
where [PASSWORD] is the password for the root user. 

### c) Create tables
Included in this repository under `setup/command_line` is a script named `setup_mysql_tables.sh` , which will run each file in the sql directory in order. Call it via
```
sh ~/ebm-code/setup/command_line/setup_mysql_tables.sh
```
Enter the MySQL root password when prompted. 

**WARNING:** running these scripts will drop any existing tables. Be sure to back up the database if it has already been used.


## 3. PAC Configuration

### a) Add QT to PATH

To add QT as an environment variable to the `PATH`, run:

```
echo 'export PATH="[PATH/TO/QT]:$PATH"' >> ~/.bash_profile
```


where [PATH/TO/QT] points to the bin directory for Qt, for example 
`/Users/ebm/Qt/5.12.3/clang_64/bin`


### b) Set up the QMYSQL Driver
Most versions of Qt do not come with the QMYSQL Driver available. Without it, the PAC Communicator will be unable to connect to the database. To check if the QMYSQL driver is present, go to the directory `/Users/ebm/Qt/[VERSION]/clang_64/plugins/sqldrivers` and look for `libqsqlmysql.dylib` (make sure to check the filename, as there are similarly named files: this one is Lib Q SQL MySQL . Dylib). If this file exists, move on to substep ii; otherwise, continue to substep i. 
>**NOTE:** There are two different directories named `sqldrivers` . Make sure the full directory path is correct. 

#### i) Compile QMYSQL to create dylib files
You will need to recompile the source in order to add the QMYSQL driver using the included 'configure' script. 
Run:
```
cd ~/Qt/[VERSION]/Src
./configure -sql-mysql
make
```
where [VERSION] is the version of Qt, ex 5.12.3 .

> **NOTE:** This step takes several hours. This is a good time to go to lunch or take a nap. *Do not continue with installation until the compiler is finished.*

Once the build completes, go into the source for MySQL and make it by running:
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




#### ii) Load QMYSQL driver

Qt needs to be told where to find MySQL. Until you pinpoint the MySQL Client Library, setup scripts and the PAC Communicator will fail with the error "QMYSQL driver not loaded."

Check what version(s) of the MySQL Client Library you have. This is a Dynamic Library (dylib) file named "libmysqlclient" found in mysql's lib directory.

Run:
```
cd /usr/local/mysql/lib
ls -a
```

You will get a list of all of the files in that directory. Check if you have a file named *exactly*:
`libmysqlclient.dylib`

If you do not have this file, there should be a file named:
`libmysqlclient.[X].dylib`

 where [X] is a version number. If this is the case, take note of the number.

If neither are present, you may need to install Connector/C using Homebrew. 
Run:
```
brew install mysql-connector-c
```
Then check again for a MySQL client library file (`libmysqlclient.dylib` or `libmysqlclient.[X].dylib`) in `/usr/local/mysql/lib`.

Once you have found the `libmysqlclient.dylib` file, check to see what library Qt is expecting. Go to the SQL Drivers directory under Plugins for the compiler (clang) you are using and run otool to list the libraries. In the Terminal, run:
```
cd ~/Qt/[VERSION]/clang_64/plugins/sqldrivers
otool -L libqsqlmysql.dylib
```
where *[VERSION]* is the version of Qt you installed (ex 5.12.3). 

This command will list the libraries that Qt's SQL Driver is looking for. The important one is `libmysqlclient`. This library should point to the exact path of the dylib file you located (either `libmysqlclient.dylib` or `libmysqlclient.[X].dylib` in `/usr/local/mysql/lib`). 

If the SQL driver is not pointing to the correct file, you will need to run install_name_tool to correct it. While still in the `~/Qt/[VERSION]/clang_64/plugins/sqldrivers` directory, run:
```
install_name_tool -change [OLD PATH] [NEW PATH] libqsqlmysql.dylib
```
Where *[OLD PATH]* is the directory currently listed by otool, and *[NEW PATH]* is the actual location of your client file. For an example:
```
install_name_tool -change @rpath/libmysqlclient.20.dylib /usr/local/mysql/lib/libmysqlclient.dylib libqsqlmysql.dylib
```


### c) Run *Printer Settings* 

Run the *Printer Settings* app inside the printer_settings directory, and choose the settings which match your current setup. If you would rather do this via terminal, you will need to manually insert a new row in the settings table via MySQL.
>**NOTE** The only printer drivers listed by this app are ones which already exist on the machine. If drivers are added later, you will need to re-run the *Printer Settings* app. 

### d) Compile the PAC Communicator
Change directories to PAC_Communicator and run:
```
qmake
make
```
This will create the PAC Communicator app under /Users/ebm. 
>**NOTE**: The PAC Comm will crash on opening if you have not saved your settings via the *Printer Settings* app (previous step) or have not installed printer drivers (step 1.H).

You can test the PAC Communicator in Island Mode (disconnected from PAC) via the following command:
```
./PAC_Communicator.app/Contents/MacOS/PAC_Communicator --island
```


If everything is set up properly, the PAC Communicator should run in Island Mode without crashing or producing errors. If you do not run in Island Mode it will hang but should not crash.


## 4. Webserver Configuration

### a) Set up Apache Server 
Apache is included with Mac OS; all you need to do is to change the configuration and start it.
There is a file in this repository under `setup/` named `httpd.conf` . Replace the file `/etc/apache2/httpd.conf` with the `httpd.conf` file found in this repository.

To start the server, run:
```
sudo apachectl start
```
You can then test the configuration by opening Safari and going to "localhost" (do not add http:// or .com). 

### b) PHP Setup
There is a file in this repository under `setup/` named `php.ini` . Replace the file `/etc/php.ini` with the `php.ini` file found in this repository.

## 5. Network/Bundle Processor Setup

For security reasons, the credentials for the network connection are not included in this repository. In order to connect to the network to pull the network catalog, you will need to add these in manually to the table "credentials". You will additionally need to provide credentials to the order_tosser script. Please confirm each of these values before setup. 

### a) In the database
To add credentials into the database, run:

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
Create a file `/bin/order_tosser.ini` that contains the following:
```
net_uname   = [EBM ID]
net_passwd  = [EBM PASSWORD]
net_authkey = [NETWORK AUTHORIZATION KEY]
net_ip      = [STORE IP ADDRESS]
```

## 6. Set MySQLdb library paths
As is the case with Qt, Python needs to know where the MySQL library is, along with some other dependencies (SSL and Crypto).
Check where Python is expecting libmysqlclient with otool. Run:
```
cd /Library/Python/[VERSION]/site-packages/MySQLdb/
otool -l _mysql.so
```
This will tell you where Python ecxpects libmysql client, as well as which versions of libssl and libcrypto it expects. If these do not match, run:
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
In order to zip/unzip data bundles from the network you need the GPG keys and passphrase. The keys look like this:

```
-----BEGIN PGP [PUBLIC/PRIVATE] KEY BLOCK-----
[hash]
-----END PGP [PUBLIC/PRIVATE] KEY BLOCK-----
```

You should have one private key and one public key, and additionally, have a short passphrase (approximately 15 bytes). 
Create a file `/opt/odb/gpg/passphrase.txt` and paste the passphrase (with no leading or trailing characters) inside.
Then, save the keys in separate txt files (location does not matter). For both the public and private key, run in terminal:

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
Now you need to set the GPG environment variable to use TTY correctly. 
To add it to PATH, run:

```
echo 'export GPG_TTY=$(tty)' >> ~/.bash_profile
```

### d) Allow automatic passphrase entry
Configure GPG-Agent to allow for automatic passphrase entry. In the file `/Users/ebm/.gnupg/gpg-agent.conf` , add the line:

```
allow-loopback-pinentry
```

Note that gpg-agent.conf may not exist; if it does not, create it.

## 8. Set up Makebook/Netmakebook

### a) Create necessary directories
The code is expecting a few extra directories which need full read, write, and execute permissions.
To create them, run:
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
Makebook and netmakebook rely on three libraries. 
To install these libraries, run:
```
brew install poppler
brew install ghostscript
brew install imagemagick
```

### c) Copy test book files into catalog

Move the five directories found under `setup/catalog` into `/opt/odb/catalog` . 

### d) Create cron jobs
run
```
crontab -e
```
then paste the contents of `setup/crontab.txt` into the editor. 
To exit the editor, press the `Esc` key, then `:` , then `w` , then `q` , then enter to save changes.



## 9. PAC Connection

The PAC communicates on `http://192.168.0.250/views/main_a.html` and must be opened in a browser window. It also requires a specific extension to allow its legacy Java applet to run. The extension is in this repository under `setup/chrome_extension`.

In chrome, go to `chrome://extensions/` and turn on Developer Mode (upper right-hand corner). Then click "Load Unpacked" (upper left-hand corner) and select the folder. 
When opening the PAC you will need to enable this extension.
