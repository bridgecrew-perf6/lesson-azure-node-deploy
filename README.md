# Deploy Node/Express App to Azure Linux VM

The following walks though deploying a Node/Express app to an Azure Linux Virtual Machine.

<br>

![cloud computing](./images/cloud-computing.jpg)

<br>

---
## Create an Express App
<br>

We'll use a bare bones Express app for this walk through. Let's set it up now!

```bash
mkdir node-azure-deploy

cd node-azure-deploy

echo node_modules > .gitignore

touch index.js

git init

npm init

npm i express
```

In `index.js` ...

```js
const express = require('express')

const app = express()
const PORT = 4000

app.get('/', (req, res) => {
  res.send('Welcome to my app')
})

app.get('/about', (req, res) => {
  res.send('All about my app')
})

app.listen(PORT, () => {
  console.log(`Server up and running on port: ${PORT}`)
})
```

Test that your app works locally in your browser by navigating to `localhost:4000`.

<br>

---
## Push up Your Code

Create a repo in Github for your project, save, commit your changes, and push up your code.

...

Alright, let's roll!

<br>

---

## Create a Cloud Init File

We'll create this in the project directory for the sake of organization, although it could be anywhere.
```bash
touch cloud-init-github.txt
```

Paste this code into `cloud-init-github.txt`
```txt
#cloud-config
package_upgrade: true
packages:
  - nginx
write_files:
  - owner: www-data:www-data
    path: /etc/nginx/sites-available/default
    content: |
      server {
        listen 80;
        location / {
          proxy_pass http://localhost:4000;
          proxy_http_version 1.1;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection keep-alive;
          proxy_set_header X-Forwarded-For $remote_addr;
          proxy_set_header Host $host;
          proxy_cache_bypass $http_upgrade;
        }
      }
runcmd:
  # install Node.js
  - 'curl -sL https://deb.nodesource.com/setup_14.x | sudo -E bash -'
  - 'sudo apt-get install -y nodejs'
  # clone GitHub Repo into myapp directory
  - 'cd /home/azureuser'
  - git clone "https://github.com/your-username/your-project-repo" myapp
  # Start app
  - 'cd myapp && npm install && npm start'
  # restart NGINX
  - systemctl restart nginx
```
**Do This**: Replace `https://github.com/your-username/your-project-repo` toward the bottom of the file with the url to your project repo deployed on Github.

---

[Cloud Init](https://cloudinit.readthedocs.io/en/latest/) is the industry standard for cross-platform cloud instance initialization. This file will tell the VM how to configure the Nginx server and what commands to run to set up our project.

Things of Note:
  - We are telling the server to listen for HTTP requests on port 80 and to direct them to localhost:4000, where are app will be running.
  - `runcmd` will specify commands to run to initialize the app.

<br>

---
## Configure PM2

[PM2](https://www.npmjs.com/package/pm2) is a production process manager for Node.js apps. We will use it to start our Express app in the Azure VM.

```bash
npm i pm2
```

In `package.json` add these four scripts. We can remove the "test" script for this example.

```json
...

  "scripts": {
    "stop": "pm2 kill",
    "start": "pm2 start index.js",
    "list": "pm2 list",
    "restart": "pm2 restart index.js"
  },

...
```

<br>

---
## Push up your Changes

Save, commit, and push up your changes to your Github project repo.

<br>

---
## Create an Azure Resource Group

[Resource groups](https://docs.microsoft.com/en-us/azure/azure-resource-manager/management/manage-resource-groups-portal#what-is-a-resource-group) in Azure give us a way to organize the various resources deployed to the cloud. You may wish to contain all of the resources for a particular application in the same group.

If you haven't already, install the [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli).

Login to the Azure CLI
```bash
az login
```

Create a resource group
```bash
az group create \
--location eastus \
--name rg-demo-vm-eastus
```

The value given for name will be the name of the resource group.

<br>

---
## Create a Virtual Machine

A [virtual machine](https://azure.microsoft.com/en-us/overview/what-is-a-virtual-machine/#overview) is an on-demand, scalable computer resource available in Azure. We will use it host our Express app.

Make sure you are navigated to your project directory, or where ever you stored your `cloud-init-github.txt` file.

Create a virtual machine
```bash
az vm create \
--resource-group rg-demo-vm-eastus \
--name demo-vm \
--location eastus \
--image UbuntuLTS \
--admin-username azureuser \
--generate-ssh-keys \
--custom-data cloud-init-github.txt
```

<br>

After running this command, if all went well you should get an output that looks something like this.

```bash
{
  "fqdns": "",
  "id": "/subscriptions/fk2kfksf914-61de-403f-b38a-234sdf03fs/resourceGroups/rg-demo-vm-eastus/providers/Microsoft.Compute/virtualMachines/demo-vm",
  "location": "eastus",
  "macAddress": "00-0D-3A-1F-E0-71",
  "powerState": "VM running",
  "privateIpAddress": "10.0.0.4",
  "publicIpAddress": "20.127.9.123",
  "resourceGroup": "rg-demo-vm-eastus",
  "zones": ""
}
```

The output includes the IP address of our deployed VM. Store this for later.

<br>

---
## Allow HTTP Requests to your Virtual Machine

If we attempt to visit this IP address now the browser will be left hanging. We haven't configured our VM to accept HTTP requests and by default this port is closed.

Open it by running ...
```bash
az vm open-port \
--port 80 \
--resource-group rg-demo-vm-eastus \
--name demo-vm
```

Wait a couple of minutes and you should now be able to view your Express app at the IP address provided!
