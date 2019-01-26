# Udacity - Linux Server Project

This is a step by step guide for installing and configuring a Linux server and preparing it to host a flask web application.

## Getting Started

Requierments for this guide:  

* Terminal (Linux, Mac) or Git Bash (Windows) - if you don't have git installed you can get it [here](git-scm.com).
* Ubuntu Instance.

## IP & Hostname

Host Name: http://ec2-3-8-117-100.eu-west-2.compute.amazonaws.com

IP Address: 3.8.117.100


## Setting up a Ubuntu instance (Amazon Lightsail)

1. Go to the [Amazon Lightsail](https://aws.amazon.com/lightsail/) website.

2. Click get started for free.

3. Click create instance.

4. Select any location you want.

5. Select OS Only and Ubuntu 16.04.

6. Scroll down and name your instance whatever you want and click create.

7. Wait for the instance to Run, once it's running click on it and scroll down to "Account Page".

8. Now click download to get your private key. The file type is .pem and will be used to SSH into the server.

9. Finally, configure the ports that Amazon Lightsail allows, to allow ports 2200 and 123.

10. Click the networking tab 

11. From this tab click add another under "Firewall" and choose Custom for application, TCP for protocol, and the port number under Port Range. Then click save.


## Linux Configuration

12. Store the SSH key ( .pem file) you downloaded from your Ubuntu instance's account page in the .ssh folder at the root of your user directory. e.g Macintosh HD/Users/[Your username]/.ssh/

13. To make your key secure type ```$ chmod 600 ~/.ssh/udacity.pem ``` into the terminal.

14. Log into the server as the user ubuntu with your key. From the terminal type ```$ ssh -i ~/.ssh/udacity.pem ubuntu@3.8.117.100 ```

15. Once logged in you will see the command line change to Ubuntu@[ip-your-private-ip]:$

16. Create a user called grader. From the command line type ```$ sudo adduser grader```. It will ask for a password and then a few other fields which you can leave blank.

17. We must create a file to give the user grader superuser privileges. To do this type ```$ sudo nano /etc/sudoers.d/grader```. 

18. Type ```grader ALL=(ALL:ALL)``` to exit hit Ctrl-X on your keyboard, type 'Y' to save, and return to save the filename.

19. Upgrade the current packages, and install new updates with these three commands:


```

$ sudo apt-get update
$ sudo apt-get upgrade
$ sudo apt-get dist-upgrade

```

20. Install a useful tool called Finger with the command ```$ sudo apt-get install finger```. This tool will allow us to see the users on this server.

21. Now we must create an SSH Key for our new user grader. From a new terminal run the command: ```$ ssh-keygen -f ~/.ssh/udacity.rsa```.

22. In the same terminal we need to read and copy the public key using the command:  ```$ cat ~/.ssh/udacity.rsa.pub. ``` Copy the key from the terminal.

23. Back in the server terminal locate the folder for the user grader, it should be /home/grader. Run the command ```$ cd /home/grader ``` to move to the folder.

24. Create a directory called .ssh with the command ```$ mkdir .ssh```.

25. Create a file to store the public key with the command ```$ sudo touch .ssh/authorized_keys```.

26. Edit that file using ```$ sudo nano .ssh/authorized_keys``` and paste in the public key.

27. We must change the permissions of the file and its folder by running

```
$ sudo chmod 700 /home/grader/.ssh
$ sudo chmod 644 /home/grader/.ssh/authorized_keys 

```

28. Change the owner of the .ssh directory from root to grader by using the command ```$ sudo chown -R grader:grader /home/grader/.ssh ```

29. The last thing we need to do for the SSH configuration is restart its service with ```$ sudo service ssh restart```.

30. Disconnect from the server.

31. Now we need to login with the grader account using ssh. From your local terminal type ```$ ssh -i ~/.ssh/udacity.rsa grader@3.8.117.100 ```.

32. We need to enforce key authentication from the ssh configuration file by editing ```$ sudo nano /etc/ssh/sshd_config```. Change PermitRootLogin and PasswordAuthentication to no. Change Port 22 to Port 2200. Exit Ctrl-X then save the file.

33. Restart ssh again: ```$ sudo service ssh restart ```.

34. Disconnect from the server.

35. Connect again but adding -p 2200 at the end. ```$ ssh -i ~/.ssh/udacity.rsa grader@3.8.117.100 -p 2200```.

36. From here we need to configure the firewall using these commands

```
$ sudo ufw allow 2200/tcp
$ sudo ufw allow 80/tcp
$ sudo ufw allow 123/udp
$ sudo ufw enable

```

That's it for Linux configuration.

## Web Application Deployment

Requirements: 

* Python virtual environment
* Apache with mod_wsgi
* PostgreSQL
* Git

37. Start by installing the required software

```
$ sudo apt-get install apache2
$ sudo apt-get install libapache2-mod-wsgi python-dev
$ sudo apt-get install git

```

38. Enable mod_wsgi with the command ```$ sudo a2enmod wsgi ``` and restart Apache using ```$ sudo service apache2 restart```. 

39. We now have to create a directory for our catalog application and make the user grader the owner

```
$ cd /var/www
$ sudo mkdir catalog
$ sudo chown -R grader:grader catalog
$ cd catalog

```

40. In this directory we will have our catalog.wsgi file ```var/www/catalog/catalog.wsgi```, our virtual environment directory which we will create soon and call ```venv /var/www/catalog/venv```, and also our application which will sit inside of another directory called catalog ```/var/www/catalog/catalog```.

41. First lets start by cloning our Catalog Application repository by ```$ git clone [repository url] catalog```.

42. Create the .wsgi file by ```$ sudo nano catalog.wsgi``` and make sure your secret key matches with your project secret key

```
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/")

from catalog import app as application
application.secret_key = 'super_secret_key'

```

43. Rename your project file in your catalog application folder to __init__.py by ```$ mv project.py __init__.py```.

44. Now lets create our virtual environment, make sure you are in ```/var/www/catalog```.

```
$ sudo pip install virtualenv
$ sudo virtualenv venv
$ source venv/bin/activate
$ sudo chmod -R 777 venv

```

45. While our virtual environment is activated we need to install all packages required for our Flask application.

```
$ sudo apt-get install python-pip
$ sudo pip install flask
$ sudo pip install httplib2 oauth2client sqlalchemy psycopg2

```

46. Now for our application to properly run we must do some tweaking to the ```__init__.py``` file.

47. Anywhere in the file where Python tries to open ```client_secrets.json``` must be changed to its complete path ex: ```$ /var/www/catalog/catalog/client_secrets.json ```

48. Time to configure and enable our virtual host to run the site

```
$ sudo nano /etc/apache2/sites-available/catalog.conf

```

Paste in the following:

```
<VirtualHost *:80>
    ServerName [Public IP]
    ServerAlias [Hostname]
    ServerAdmin admin@35.167.27.204
    WSGIDaemonProcess catalog python-path=/var/www/catalog:/var/www/catalog/venv/lib/python2.7/site-packages
    WSGIProcessGroup catalog
    WSGIScriptAlias / /var/www/catalog/catalog.wsgi
    <Directory /var/www/catalog/catalog/>
        Order allow,deny
        Allow from all
    </Directory>
    Alias /static /var/www/catalog/catalog/static
    <Directory /var/www/catalog/catalog/static/>
        Order allow,deny
        Allow from all
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>

```

*To find your server's hostname go [here](https://whatismyipaddress.com/ip-hostname) and paste the IP address.*

Save and quit nano.

49. Enable to virtual host: ```$ sudo a2ensite catalog.conf ``` and DISABLE the default host ```$ a2dissite 000-default.conf``` otherwise your site will not load with the hostname.

50. The final step is setting up the database

```
$ sudo apt-get install libpq-dev python-dev
$ sudo apt-get install postgresql postgresql-contrib
$ sudo su - postgres -i
$ psql

```

51. Create a database user and password

```
postgres=# CREATE USER catalog WITH PASSWORD [password];
postgres=# ALTER USER catalog CREATEDB;
postgres=# CREATE DATABASE catalog with OWNER catalog;
postgres=# \c catalog
catalog=# REVOKE ALL ON SCHEMA public FROM public;
catalog=# GRANT ALL ON SCHEMA public TO catalog;
catalog=# \q
$ exit


```

Your command line should now be back to ```grader```.


52. Restart your apache server ```$ sudo service apache2 restart ``` and now your IP address and hostname should both load your application.


## References:

* https://github.com/mulligan121/Udacity-Linux-Configuration
* https://github.com/iliketomatoes/linux_server_configuration
* https://github.com/kongling893/Linux-Server-Configuration-UDACITY

