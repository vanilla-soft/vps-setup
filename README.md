## First Time VPS Setup for Nodejs/Nextjs Production with Nginx, PM2 and MongoDB with SSL

This guide will show you how you can deploy a full-stack NextJS application to your own vps server. I'm using a vps server from Hostinger. You can also deploy any NodeJS application following this tutorial.

<!-- ### Prerequisites -->

### Server Connection

*connect to vps via ssh as root user*
```
ssh root@your-server-ip
```
<!-- `ssh root@62.72.56.151` -->

### Necessary Package Installation

*install nvm (node version manager)*

```
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.5/install.sh | bash
```

```
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"  # This loads nvm
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"  # This loads nvm bash_completion
```

*install node (i'm using node version 18.18, you can use any version you need)*

```
nvm install 18.18.0
```

*install yarn globally (optional, you can use npm as well)*
```
npm i -g yarn
```

### GitHub Configuration (optional)

I'm using GitHub to fetch my project. So, I like to add a ssh key to GitHub so that GitHub can recognize this device and do not ask for password everytime.

*generate ssh key*
```
ssh-keygen -t ed25519 -C “your-github-email”
eval "$(ssh-agent -s)" 
ssh-add ~/.ssh/id_ed25519 
cat < ~/.ssh/id_ed25519.pub
```

<!-- ```
ssh-keygen -t ed25519 -C “creativestudio.soft@gmail.com”
eval "$(ssh-agent -s)" 
ssh-add ~/.ssh/id_ed25519 
cat < ~/.ssh/id_ed25519.pub
``` -->

*then copy the key and add it to GitHub [`see here`](https://www.example.com)*

**create any directory to keep your projects (I named mine `applications`)**

```
mkdir applications  # create directory
cd applications/    # go to directory
```

**clone project from git and install dependencies**
```
git clone git_ssh_repo_url project_folder_name  # clone project from github
cd project_folder_name    # go to project directory
yarn                      # install dependencies
```
**create and update environment file (optional)**

```
cp .env.local.example .env.local    # this is for nextjs 13
```

*if you don't have an example file, create one or modify existing file*

```
nano .env.local
```

**build your application (optional)**
As I'm deploying a NextJS application and I'm going to deploy it as production, I need to build it first.

```
yarn build
```

### Setup PM2 and Run Your Application
*install pm2 globally*

```
npm i -g pm2
```

**my scripts inside package.json file**
Make sure you defined the port where you wish to run your application.

```
"scripts": {
    "dev": "next dev -p 3000",
    "build": "next build",
    "start": "next start -p 3000",
    "lint": "next lint"
  },
```

**start your app using pm2**

```
pm2 start yarn --name “application_name” -- start   # production
```

**if you don't use a nextjs app and you have an application startup file, you can run the following command**

```
pm2 start app.js --name “application_name”   # app.js - your startup file name
```

**running this command once will start all pm2 saved instance when the server reboot**

```
pm2 startup
```

**some necessary pm2 commands (run if you need)**

```
pm2 stautus
pm2 list
pm2 logs
pm2 logs index
pm2 restart all
pm2 stop index
pm2 start index
pm2 delete index
pm2 save
```

If your application doesn't have any error, it will run in the background until you stop it. You can check your application by hitting ***your_server_id:application_port*** to your browser.

### MongoDB Configuration (optional for other db user)

*install mongodb*

```
curl -fsSL https://pgp.mongodb.com/server-7.0.asc | \
   sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg \
   --dearmor

echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
```

```
sudo apt-get update
sudo apt-get install -y mongodb-org
```

```
echo "mongodb-org hold" | sudo dpkg --set-selections
echo "mongodb-org-database hold" | sudo dpkg --set-selections
echo "mongodb-org-server hold" | sudo dpkg --set-selections
echo "mongodb-mongosh hold" | sudo dpkg --set-selections
echo "mongodb-org-mongos hold" | sudo dpkg --set-selections
echo "mongodb-org-tools hold" | sudo dpkg --set-selections
```

*run mongodb*

```
sudo systemctl start mongod
```

*ensure that MongoDB will start following a system reboot*

```
sudo systemctl enable mongod
```

**configure mongodb to access it remotely**

*open mongodb config file*

```
sudo nano /etc/mongod.conf
```

*find and comment or remove the following line*
`bindIp: 127.0.0.1`

*then add the following line to access it from any ip address*
```
bindIpAll: true
```

**enable access control to protect your database**

*open mongodb shell*

```
mongosh
```

*create an admin user to `admin` database*

```
use admin
db.createUser(
  {
    user: "username",
    pwd: "user_password", // or cleartext password
    roles: [
      { role: "userAdminAnyDatabase", db: "admin" },
      { role: "readWriteAnyDatabase", db: "admin" }
    ]
  }
)
```

*restart mongodb*
```
db.adminCommand( { shutdown: 1 } )
```

To know more about MongoDB user and role management, follow MongoDB [documentation](https://www.mongodb.com/docs/manual/core/authorization/).

*again open mongodb config file*

```
sudo nano /etc/mongod.conf
```

*add the security.authorization configuration file setting*

```
security:
    authorization: enabled
```

*restart mongodb*

```
sudo systemctl restart mongod
```

### Nginx Configuration

*install nginx*

```
sudo apt update
sudo apt install nginx
```

*open nginx config file*

```
sudo nano /etc/nginx/sites-available/default
```

*find default server configuration and update it*

```
server_name yourdomain.com www.yourdomain.com;

location / {
    proxy_pass http://localhost:your_application_port;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
    proxy_cache_bypass $http_upgrade;
}
```

*if you want to serve multiple applications, you need to add more server like following*

```
server {
    # root /var/www/html;
    index index.html index.htm;

    server_name your_another_domain.com www.your_another_domain.com;

    location / {
            proxy_pass http://localhost:3002;   # whatever port your app runs
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
    }
}
```

### SSL Certification (using certbot)

*install certbot*
```
sudo snap install --classic certbot
```

*prepare the Certbot command*
```
ln -s /snap/bin/certbot /usr/bin/certbot
```

*run certbot*
```
certbot --nginx
```

*test automatic renewal*
```
certbot renew --dry-run
```

<!-- ### What is not covered in this tutorial -->