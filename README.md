# Deploying Catalog App on AWS Lightsail
Author: Ellen Liu
Last Edit: 10/30/17

In this project, we will be deploying a flask application, the catalog app, on to AWS Lightsail. We will be using Ubuntu 16.04 LTS.

[Catalog App](http://ec2-34-215-197-22.us-west-2.compute.amazonaws.com/)
[Catalog Github](https://github.com/ellenmliu/Catalog)
IP: 34.215.197.22 Port 2200

# Set up AWS Lightsail
1) Sign up or log into [AWS Lightsail](https://lightsail.aws.amazon.com/).
2) Click on 'Create Instance'
3) Select 'Linux/Unix'
4) Click on 'OS Only' and select 'Ubuntu'
5) Select a plan and name your instance. Click 'Create' when done

# Set up Access to Instance via Terminal
We need to retrieve our SSH key pair to connect via terminal. We want to do this before changing the ports so we can log in. You can use the 'Connect using SSH' for now, but once the ports are changed, we won't be able to use it because it uses the default port.

1) Go to 'Account' and click on 'SSH keys'
2) There should be a default key. Go ahead and download it.
3) In terminal, `cd` to the directory where you saved the file to
4) We need to change permissions to it, so run `cdmod 600 <LIGHTSAILFILENAME>`
5) Run `ssh USER@PUBLICIP -i <LIGHTSAILFILENAME>` (You can find the USER and PUBLICIP on Lightsail Instance page)

# Update the Server
``` sudo apt-get update
    sudo apt-get upgrade
```

# Secure the Server
We will be changing the ports. Must do this in order or else you will be locked out of the server.
1) Configure Lightsail firewall. To do so, go to the instance page and click on 'Nextworking'. There should be a Firewall section and 'add another'. Select Custom, TCP and the port range is 2200. Go ahead and save.
2) On ubuntu server in terminal, run `sudo nano /etc/ssh/sshd_config` and change `Port 22` to `Port 2200`.
3) Run `sudo service ssh restart`. Now when you login you use ssh `ssh USER@PUBLICIP -i <LIGHTSAILFILENAME> -p 2200`

# Configure Firewall Rules
We want to only allow incoming connections for SSH(2200), HTTP(80), and NTP(123).
1) Add the following to add those rule
```sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2200/tcp
sudo ufw allow www
sudo ufw allow 123/udp
```
You can use `sudo ufw status` to see if the ufw is active and `sudo ufw show added` to check the list before enabling.
2) Run `sudo ufw enable` to enable the set rules and check the status to see if all of the rules are correct.

Check if you can login in. Run `exit` and then `ssh USER@PUBLICIP -i <LIGHTSAILFILENAME> -p 2200`. If you see `*** System restart required ***`, then run `sudo reboot` and log back in.

# Creating a New User
Still logged in the instance, we will be creating a new user, grader.
1) Run `sudo adduser grader`. Give it a password.
2) To give it sudo access, we will create a file by running `sudo nano /etc/sudoers.d/grader` and write `grader ALL=(ALL) NOPASSWD:ALL` in the file

# Create SSH for Grader
In order to log in with the user we created, we need a SSH key pair.
1) On your local machine, run `ssh-keygen`. It will ask where to save the key, so match with the directory of .ssh and then name the file however you want. You can enter in any passphrase for it or leave it blank.
2) It will generate 2 files. The `.pub` file will be placed on the server. Go ahead and login as the grader. If you are logged in as ubuntu, run `sudo login grader` and provide the password. Make a directory that will for SSH
``` mkdir .ssh
    cd .ssh
```
3) Create authorized_keys file using `sudo nano authorized_keys`. On your local machine, run `cat .ssh/<FILENAME.pub>` and copy that. Then paste the content into your authorized_keys file on the server page.
4) Now we need to set the permissions.
``` sudo chown grader:grader /home/grader/.ssh
    sudo chmod 700 /home/grader/.ssh
    sudo touch .ssh/authorized_keys
    sudo chown grader:grader /home/grader/.ssh/authorized_keys
    sudo chmod 600 /home/grader/.ssh/authorized_keys
```
5) To check your work, try to login in to grader using the command `ssh grader@PUBLICIP -i ~/.ssh/<FILENAME> -p 2200`

# Change Timezone to UTC
Run `sudo timedatectl set-timezone UTC` and run `date` to check the timezone.

# Set up Apache
1) We will set it up using `sudo apt-get install apache2`. If you paste in the IP address into your browser, you should see the Apache2 Ubuntu Default page letting you know that it works.
2) Run `sudo apt-get install libapache2-mod-wsgi` so Apache can serve a Python mod_wsgi application.

# Install and Configure PostgreSQL
1) Install PostgreSQL by running `sudo apt-get install postgresql`
2) Run `sudo nano etc/postgresql/9.5/main/pg_hba.conf` and 'under IPv4 local connections' it should have `127.0.0.1/32` and under 'IPv6 local connections' it should have `::1/128`. This prevents remote connects to PostgreSQL.
3) To create a PostgreSQL user, we will run `sudo -u postgres createuser -P catalog` and set a password for it. The user has limited permissions.
4) Create an empty 'catalog' database using `sudo -u postgres createdb -O catalog catalog`.

# Install git and Setup Application
1) Install using `sudo apt-get install git`.
2) Change directory to `/var/www`
3) Create the folder and give ownership to www-data using
``` sudo mkdir catalogapp
    sudo chown www-data:www-data catalogapp
```
4) Clone repository into there. I did `sudo -u www-data git clone https://github.com/ellenmliu/Catalog.git catalogapp`
5) Change directory into that folder and create a new wsgi file. I called mine catalogapp.wsgi
```
import sys
sys.path.insert(0,'/var/www/catalogapp')

from views import app as application
```
6) Go on the developer pages on Google and Facebook to add in the IP and the AWS assigned URL. For the Google json file we need to find the `javascript_origins` and add in the two addresses.
7) Configure by creating a file in `/etc/apache2/sites-available` called `catalogapp.conf` and put in
```
<VirtualHost *:80>
        WSGIScriptAlias / /var/www/catalogapp/catalogapp.wsgi
        <Directory /var/www/catalogapp/>
                Order allow,deny
                Allow from all
        </Directory>
        ErrorLog ${APACHE_LOG_DIR}/error.log
        LogLevel info
        CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
8) Right now it is still running the default so we need to disable it using `sudo a2dissite 000-default.conf`
9) Enable the catalogapp using `sudo a2ensite catalogapp.conf` and use `sudo service apache2 reload`
10) Run
```
curl https://bootstrap.pypa.io/ez_setup.py -o - | sudo python
sudo easy_install pip
sudo pip install requests
sudo pip install httplib2
sudo pip install flask
sudo pip install sqlalchemy
sudo pip install oauth2client
sudo pip install psycopg2
```
11) Make some changes in `views.py`. Move the `app.secret_key = 'super_secret_key'` outside from the main and put it global. Then change the engine to be `engine = create_engine('postgresql://catalog:catalog@localhost/catalog')` so that it can connect with the catalog database. Change the CLIENT_ID to `CLIENT_ID = json.loads(
25	    open('client_secrets.json', 'r').read())['web']['client_id']
26	    open('/var/www/catalogapp/client_secrets.json', 'r').read())['web']['client_id']`. In the function gconnect(), need to change line to `oauth_flow = flow_from_clientsecrets('/var/www/catalogapp/client_secrets.json', scope='')` and in fbconnect(), need to change to  `json_loaded = json.loads(open('/var/www/catalogapp/fb_client_secrets.json', 'r').read())`. Change the last line to `app.run()`.
12) In `database_setup.py` and `categoriesanditems.py`, we need to connect to the catalog database by changing to `engine = create_engine('postgresql://catalog:catalog@localhost/catalog')`.
13) Run
```
python database_setup.py
python categoriesanditems.py
```
14) On your browser, you can either go on the IP address or the AWS given URL.

# Third Party Sources
1) https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/managing-users.html
2) https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps
3) http://alex.nisnevich.com/blog/2014/10/01/setting_up_flask_on_ec2.html
4) https://help.ubuntu.com/community/PostgreSQL
5) https://www.a2hosting.com/kb/developer-corner/apache-web-server/viewing-apache-log-files
