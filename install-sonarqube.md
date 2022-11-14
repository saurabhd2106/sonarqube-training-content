# Installing Sonarqube on an ubuntu machine

Pre-requite --

One Ubuntu 18.04 server with 3GB or more memory set up by following this Initial Server.
Oracle Java 11 installed on the server. Check the version of the java by running a command "java --version"
Nginx installed as a server
MySQL installed as a database

# Step 1 — Preparing for the Install

You need to complete a few steps to prepare for the SonarQube installation. As SonarQube is a Java application that will run as a service, and because you don’t want to run services as the root user, you’ll create another system user specifically to run the SonarQube services. After that, you’ll create the installation directory and set its permissions, and then you’ll create a MySQL database and user for SonarQube.

First, create the sonarqube user:

```
sudo adduser --system --no-create-home --group --disabled-login sonarqube
```

This user will only be used to run the SonarQube service, so this creates a system user that can’t log in to the server directly.

Next, create the directory to install SonarQube into:

```
sudo mkdir /opt/sonarqube
```

SonarQube releases are packaged in a zipped format, so install the unzip utility that will allow you to extract those files

```
sudo apt-get install unzip
```

Next, you will create a database and credentials that SonarQube will use. Log in to the MySQL server as the root user:

```
sudo mysql -u root -p
```

Then create the SonarQube database:

```
CREATE DATABASE sonarqube;
```

Now create the credentials that SonarQube will use to access the database.

```
CREATE USER sonarqube@'localhost' IDENTIFIED BY 'some_secure_password';
```

Then grant permissions so that the newly created user can make changes to the SonarQube database:

```
GRANT ALL ON sonarqube.* to sonarqube@'localhost';
```

Then apply the permission changes and exit the MySQL console:

```
FLUSH PRIVILEGES;
EXIT;

```

Now that you have the user and directory in place, you will download and install the SonarQube server.

# Step 2 — Downloading and Installing SonarQube

Start by changing the current working directory to the SonarQube installation directory:

```
cd /opt/sonarqube
```

Download the latest version of Sonarqube 

```
sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.7.1.62043.zip
```

Unzip the file:

```
sudo unzip sonarqube-9.7.1.62043.zip
```

update the permissions so that the sonarqube user will own these files, and be able to read and write files in this directory:

```
sudo chown -R sonarqube:sonarqube /opt/sonarqube
```

# Step 3 — Configuring the SonarQube Server

We’ll need to edit a few things in the SonarQube configuration file. Namely:

1. We need to specify the username and password that the SonarQube server will use for the database connection.
2. We also need to tell SonarQube to use MySQL for our back-end database.
3. We’ll tell SonarQube to run in server mode, which will yield improved performance.
4. We’ll also tell SonarQube to only listen on the local network address since we will be using a reverse proxy.

Start by opening the SonarQube configuration file:

```
sudo nano sonarqube-9.7.1.62043/conf/sonar.properties
```

First, change the username and password that SonarQube will use to access the database to the username and password you created for MySQL:

```
    ...

    sonar.jdbc.username=sonarqube
    sonar.jdbc.password=some_secure_password

    ...
```

Next, tell SonarQube to use MySQL as the database driver:


```
    ...

    sonar.jdbc.url=jdbc:mysql://localhost:3306/sonarqube?useUnicode=true&characterEncoding=utf8&rewriteBatchedStatements=true&useConfigs=maxPerformance&useSSL=false

    ...

```

As this instance of SonarQube will be run as a dedicated server, we could add the -server option to activate SonarQube’s server mode, which will help in maximizing performance.

Nginx will handle the communication between the SonarQube clients and your server, so you will tell SonarQube to only listen to the local address.

```
...

    sonar.web.javaAdditionalOpts=-server
    sonar.web.host=127.0.0.1

...

```

Once you have updated those values, save and close the file.

Next, you will use Systemd to configure SonarQube to run as a service so that it will start automatically upon a reboot.

Create the service file:

```
sudo nano /etc/systemd/system/sonarqube.service

```

Add the following content to the file which specifies how the SonarQube service will start and stop:

```

[Unit]
Description=SonarQube service
After=syslog.target network.target

[Service]
Type=forking

ExecStart=/opt/sonarqube/sonarqube-9.7.1.62043/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/sonarqube-9.7.1.62043/bin/linux-x86-64/sonar.sh stop

User=sonarqube
Group=sonarqube
Restart=always

[Install]
WantedBy=multi-user.target

```

Close and save the file, then start the SonarQube service:

```
sudo service sonarqube start
```

Check the status of the SonarQube service to ensure that it has started and is running as expected:

```
service sonarqube status
```

Next, configure the SonarQube service to start automatically on boot:

```
sudo systemctl enable sonarqube
```

At this point, the SonarQube server will take a few minutes to fully initialize. You can check if the server has started by querying the HTTP port:

```
curl http://127.0.0.1:9000
```

# Step 4 — Configuring the Reverse Proxy

Now that we have the SonarQube server running, it’s time to configure Nginx, which will be the reverse proxy and HTTPS terminator for our SonarQube instance.

Start by creating a new Nginx configuration file for the site:

```
sudo nano /etc/nginx/sites-enabled/sonarqube
```

Add the following configuration so that Nginx will route incoming traffic to Sonarqube

```

server {
    listen 80;
    server_name sonarqube.example.com;

    location / {
        proxy_pass http://127.0.0.1:9000;
    }
}

```

Save and close the file.

Next, make sure your configuration file has no syntax errors:

```
sudo nginx -t
```

If you see errors, fix them and run sudo nginx -t again. Once there are no errors, restart Nginx:

```
sudo service nginx restart
```

For a quick test, you can now visit http://sonarqube.example.com in your web browser. You’ll be greeted with the SonarQube web interface.
