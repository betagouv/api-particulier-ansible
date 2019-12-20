# Api Particulier Ansible

Dépôt Ansible pour déployer API Particulier

## Development dependencies

- VirtualBox \^5.2.10
- Vagrant \^2.1.1
- NFS
- Ansible 2.5.15
- dnspython

## Install

### Dependencies setup

Install VirtualBox with apt:

```bash
sudo apt install virtualbox virtualbox-ext-pack
```

Install Vagrant with dpkg:

```bash
wget https://releases.hashicorp.com/vagrant/2.1.1/vagrant_2.1.1_x86_64.deb
sudo dpkg -i vagrant_2.1.1_x86_64.deb
```

Install NFS:
```bash
sudo apt install nfs-kernel-server nfs-common
```
Then reboot your machine.

Install Ansible with pip:

```bash
sudo pip install ansible==2.5.15
```

Install dnspython with pip:

```bash
sudo pip install dnspython==1.15.0
```

### Api Particulier local provisioning

Clone the repo:

```bash
git clone --recursive git@github.com:betagouv/api-particulier-ansible.git
cd api-particulier-ansible/
git submodule foreach git fetch
git submodule foreach git pull origin master
git submodule foreach git checkout master
```

Add the following hosts in `/etc/hosts`:

```text
192.168.56.25 particulier-development.api.gouv.fr
192.168.56.27 monitoring.particulier-development.api.gouv.fr
192.168.56.25 gateway-development.particulier-infra.api.gouv.fr
192.168.56.35 app1-development.particulier-infra.api.gouv.fr
192.168.56.45 app2-development.particulier-infra.api.gouv.fr
192.168.56.27 monitoring-development.particulier-infra.api.gouv.fr
```

**If you are using macOS.**
- The host's `/etc/hosts` configuration file may not take effect in the guest machines. You might need to also alter the guest machine's `/etc/hosts` after running vagrant up.
- Change the lookup function in `roles/ufw/tasks/main.yml`

```yml
- name : Allow all access to kong for apps
  ufw: rule=allow from=192.168.56.25 proto=tcp port=4443
  tags: ufw 
```
**End of macOS specific instructions.**

Then run:

```bash
vagrant up && vagrant up monitoring
ansible-galaxy install -r requirements.yml
ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -i inventories/development/hosts configure.yml
```

### Development deployment

Deploy the application manually.

On the **gateway** server:
```bash
vagrant ssh gateway
cd /opt/apps/api-particulier-auth/current
npm i
sudo systemctl restart api-particulier-auth
exit
```

On the **app1** server:
```bash
vagrant ssh app1
cd /opt/apps/api-particulier/current
npm i
sudo systemctl restart api-particulier
cd /opt/apps/svair-mock/current
npm i
sudo systemctl restart svair-mock
exit
```

On the **app2** server:
```bash
vagrant ssh app2
sudo systemctl restart api-particulier
sudo systemctl restart svair-mock
exit
```

On the **monitoring** server:
```bash
vagrant ssh monitoring
cd /opt/apps/api-stats-elk/current
npm i
sudo systemctl restart api-stats-elk
exit
```

### Test the local installation

To test that the installation went OK, we will now create a token and test it.

Token are created with signup.

> TODO create a curl script to create token manually.

> TODO create a test instance of auth.api.gouv.fr to enable token generation with the following instruction

Connect to https://particulier-development.api.gouv.fr/admin.

Click on the newly created token. Then click on "Generate new API Key".

At this step your API Key is displayed. Mind that this key is encrypted and can not be displayed anymore. Copy it, and replace USE_YOUR_OWN in `postman/development.postman_environement.json` with it.

Import your local version of `development/local.postman_environement.json` in [postman](https://www.getpostman.com/).

Import `postman/api-particulier.postman_collection.json` in postman.

You should be able to retrieve data from the caf famille json route: {{dev_host}}/api/caf/famille?numeroAllocataire={{numeroAllocataire}}&codePostal={{codePostal}}

Note that fake data can be found [here](https://github.com/betagouv/svair-mock/blob/f2c26f70eb985b44a97d1e4bab8bdee8c0439223/data/seed.csv) and [here](https://github.com/betagouv/api-particulier/blob/1fc0a91cf07d041ce8d21f23f4288ca077b81bd6/api/caf/fake-responses.json).

### Test the local monitoring

> Note that the monitoring VM must be started explicitly with `vagrant up monitoring` and will not be started with `vagrant up`.
> It is more convenient as it is not used quite often in development mode whereas it demands lots of RAM and CPU to run.

Go to https://monitoring.particulier-development.api.gouv.fr (credentials: octo|thereisabetterway).

Create an index pattern `filebeat-*` and select `@timestamp` as `Time Filter field name`.

Make an api call with postman.

From now on, your request to api-particulier are logged in Kibana. You should see them in the discover section.

You can also test the api-stats-elk api at:

https://monitoring.particulier-development.api.gouv.fr/api/stats/

You can find more info on the [api-particulier repository](https://github.com/betagouv/api-particulier)

You can install the signup application [here](https://github.com/betagouv/signup.api.gouv.fr-docker).

### Run app manually (optional)

After a first successful development deployment, you may want to run the app you are working on interactively.

API Particulier auth:
```bash
vagrant ssh gateway
sudo systemctl stop api-particulier-auth
cd /opt/apps/api-particulier-auth/current/
export $(cat /etc/api-particulier-auth.conf | xargs)
npm start
```

API Particulier:
```bash
vagrant ssh app1
sudo systemctl stop api-particulier
cd /opt/apps/api-particulier/current
export $(cat /etc/api-particulier/api-particulier.conf | xargs)
npm start
```
```bash
vagrant ssh app2
sudo systemctl stop api-particulier
cd /opt/apps/api-particulier/current
export $(cat /etc/api-particulier/api-particulier.conf | xargs)
npm start
```

Svair mock:
```bash
vagrant ssh app1
sudo systemctl stop svair-mock
cd /opt/apps/svair-mock/current
export $(cat /etc/svair-mock/svair-mock.conf | xargs)
npm start
```
```bash
vagrant ssh app2
sudo systemctl stop svair-mock
cd /opt/apps/svair-mock/current
export $(cat /etc/svair-mock/svair-mock.conf | xargs)
npm start
```

### Production-like deployment (optional)

For development purpose you may want to have a local iso-production application running. You can do it by running the deployment script instead of processing to a development deployment:

```bash
ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -i inventories/development/hosts deploy.yml
```

## Global architecture

> TODO split architecture schema in 2 parts : one for signup, one for api particulier

[![architecture](https://docs.google.com/drawings/d/e/2PACX-1vTZql6aJMbkmMiIxRy89SFPch5K-tTNIXVBv1ElXhpESRp43dSRGALdRi3ZNYsf5JlbukIN70HQv5RQ/pub?w=960&h=720)](https://docs.google.com/drawings/d/1p-v88uBrFbKMBLRKEmsrSeNWprJqnzsy08SBrQx6U4c/edit?usp=sharing)

## Manual generation of the token

### 1. Generate the api key

We manually execute the [token generation script](https://github.com/betagouv/api-particulier-auth/blob/master/src/utils/api-key.js) with any local node installation.

```bash
node
```
```js
const crypto = require('crypto');
// apiKey is shorten than production api keys to be more easy to use
const apiKey = crypto.randomBytes(16).toString('hex');
const hash = crypto.createHash('sha512').update(apiKey).digest('hex');
console.log(apiKey);
console.log(hash);
```

### 2. Create token in api-particulier-auth

Connect to gateway-development. Then:

```bash
mongo
```
```mongodb
use api-particulier
db.tokens.insert({
  name: "Mairie de Test",
  email: "contact@mairiedetest.fr",
  hashed_token: "<REPLACE WITH HASH FROM PREVIOUS STEP>",
  scopes: [ "dgfip_avis_imposition", "cnaf_attestation_droits" ]
})

// find the id of your token with
db.tokens.find()

db.tokens.update(
  { _id: ObjectId("<THE ID OF THE NEW TOKEN>") },
  { $set: { "signup_id" : "no-signup-<THE ID OF THE NEW TOKEN>" } }
)
```

You can now use the new API key.

## Generate token for api-particulier-auth api admin

Signup will push information to api-particulier-auth when a contract is validated.
To securely push data, signup will authenticate on api-particulier-auth with an API Key.
The following lines describe how to setup such a key.

First generate a random string and store it in 'api_particulier_api_key' in 'signup-ansible/inventories/development/group_vars/signup.yml'.

Then deploy the new configuration with:
```bash
ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -i inventories/development/hosts configure.yml -t signup-back
```

Hash it with bcrypt:
```bash
cd api-particulier-auth
node --require bcryptjs
```
```js
const bcrypt = require('bcryptjs')
const salt = bcrypt.genSaltSync(10);
const hash = bcrypt.hashSync("your-random-string-here", salt);
console.log(hash);
```

Store the output in 'hashed_signup_api_key' in 'api-particulier-ansible/inventories/development/group_vars/gateway.yml'.

Eventually, deploy the new configuration with:
```bash
ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -i inventories/development/hosts configure.yml -t api-particulier-auth
```

### Generate new certificate

Change to certificates directory:

```bash
cd certificates
```

Generate certificate for both app and monitoring domain. Run the following script and follow the outputted instructions:

```bash
./generate.sh
```

Encrypt the certificates:
```bash
cd ..
ansible-vault encrypt certificates/development/*
```

Run the configuration scripts:

```bash
ansible-playbook -i inventories/development/hosts configure.yml
```

## Troubleshooting

### If ansible dig lookup is failing (ufw: "Allow all access to kong for apps" task)

Install DNSmasq to provide those in your name server

```bash
sudo apt install dnsmasq
```

check your /etc/nsswitch.conf. It should look like this :

```text
passwd:         compat
group:          compat
shadow:         compat
gshadow:        files

hosts:          files mdns4_minimal [NOTFOUND=return] resolve [!UNAVAIL=return] dns
networks:       files

protocols:      db files
services:       db files
ethers:         db files
rpc:            db files

netgroup:       nis
```
