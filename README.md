# hacktrick-web-docker

My Example Docker Compose Setup I use for developing and quickly deploying a Wordpress site with Redis Cache. In this example I will be deploying the following apps as containers:

Traefik Reverse Proxy
Wordpress
Redis
MariaDB
Adminer
Portainer

Feel free to play around with the docker-compose.yml and mix and match different applications but leave the Traefix container information intact.

## Requirements

Ubuntu 20.04 (this may work with other Linux Distributions but has only been tested with Ubuntu)
Docker and Docker Compose

## Install Docker

First, update your existing list of packages:

`sudo apt update`

Next, install a few prerequisite packages which let apt use packages over HTTPS:

`sudo apt install apt-transport-https ca-certificates curl software-properties-common`

Then add the GPG key for the official Docker repository to your system:

`curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -`

`sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"`

Finally, install Docker:

`sudo apt install docker-ce`

Docker should now be installed, the daemon started, and the process enabled to start on boot. Check that it’s running:

`sudo systemctl status docker`

If you want to avoid typing sudo whenever you run the docker command, add your username to the docker group:

`sudo usermod -aG docker ${USER}`

To apply the new group membership, log out of the server and back in, or type the following:

`su - ${USER}`

Confirm that your user is now added to the docker group by typing:

`groups`

If you need to add a user to the docker group that you’re not logged in as, declare that username explicitly using:

`sudo usermod -aG docker username`

Using docker consists of passing it a chain of options and commands followed by arguments. The syntax takes this form:

`docker [option] [command] [arguments]`

## Configuring Traefik

First, we need to create a configuration file and set up an encrypted password so we can access the monitoring dashboard.

We will use the htpasswd utility to create this encrypted password.

Install the utility with the following command:

`$ sudo apt-get install apache2-utils`

Then we can generate the password with htpasswd.
And substitute secure_password with the password we like to use for the Traefik admin user:

`$ htpasswd -nb admin secure_password`

We will get an output as shown below:

`admin:$apr1$ruca84Hq$mbjdMZBAG.KWn7vfN/SNK/`

We must copy the output we receive as we will use it in the Traefik configuration file to set up HTTP Basic Authentication for the Traefik health check and the monitoring dashboard.

Next we will create a configuration file called traefik.toml using the TOML format.

We will use three of Traefik’s available providers: api, docker, and acme. The last of these, acme supports TLS certificates using Let’s Encrypt.

Open the new file text editor:

`$ nano traefik.toml`

First, we need to add two named entry points, http and https,  to which all backends have access by default:

`defaultEntryPoints = ["http", "https"]`
Next,  access to a dashboard interface and paste the output from the htpasswd command:

`...`
`[entryPoints]`
`[entryPoints.dashboard]`
`address = ":8080"`
`[entryPoints.dashboard.auth]`
`[entryPoints.dashboard.auth.basic]`
`users = ["admin:your_encrypted_password"]`
`[api]`
`entrypoint="dashboard"`

We set the dashboard to run on port 8080.

We have defined our first entryPoint.

The entryPoints section configures the addresses that Traefik and the proxied containers get listed on.

`...`
`[entryPoints.http]`
`address = ":80"`
`[entryPoints.http.redirect]`
`entryPoint = "https"`
`[entryPoints.https]`
`address = ":443"`
`[entryPoints.https.tls]`
`...`
The HTTP entry point handles port 80, while the HTTPS entry point uses port 443 for TLS/SSL.

We automatically redirect all of the traffic on port 80 to the HTTPS entry point to force secure connections for all requests.

Next change your email

`...`
`[acme]`
`email = "your_email@your_domain"`
`storage = "acme.json"`
`entryPoint = "https"`
`onHostRule = true`
`[acme.httpChallenge]`
`entryPoint = "http"`

Finally, we will configure the docker provider by adding in your domain:

`...`
`[docker]`
`domain = "your_domain"`
`watch = true`
`network = "web"`

The docker provider enables Traefik to act as a proxy in front of Docker containers.

Save the file and exit the editor.

## Running the Docker Stack via Docker Compose

Next, we need to create a Docker network for the proxy to share with containers.

The following command can be used:

`$ docker network create web`

When the Traefik container starts, we will add it to this network.
Then we can add additional containers to this network for Traefik to proxy to.

Create an empty file that will hold our Let’s Encrypt information.
We will share this into the container by using the following command:

`$ touch acme.json`

The root user inside of the container must have unique read and write access to it so that Traefik can use it.

We can change the permission using the following command:

`$ chmod 600 acme.json`

Once the file moves to Docker, the owner gets changed to root automatically inside the container.

Finally, we will change the variables in the .env file to match your environment and to set your own secure wordpress database user name and password

`APP_HOST=changeme`

`DATA=../data`

`WP_DB_NAME=changeme`
`WP_DB_USER=changeme`
`WP_DB_PASSWORD=changeme`
`WP_DB_PREFIX=wp`
`WP_DB_ROOT_PASSWORD=changeme`
`WP_DB_HOST=mariadb`

Now that the variables are set we can run the containers using docker-compose:

`docker-compose up -d`

We can access the monitoring dashboard by pointing the browser to [https://monitor.your_domain]

By using the admin login credentials we will be able to see the dashboard. But we can see contents only after adding containers.

You can monitor and manage your containers by pointing the browser to [https://containers.your_domain]

You can monitor and manage your databases by pointing the browser to [https://db.your_domain]

Finally Setup your new Wordpress site by going to [https://your_domain]

Make sure you have the relevant A records set in your DNS
