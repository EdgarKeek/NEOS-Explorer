NEOS Blockchain Explorer - 1.7.4.1
================

An open source block explorer written in node.js with masternodes and ssl support.  
See it live : https://shroud.mastermine.eu/  

This explorer is based on Iquidus Explorer (https://github.com/iquidus/explorer).  
This explorer's layout is based on MNOS Explorer v1.0.0 (https://github.com/MNOSIO/explorer).

### Pre-requisites

*  VPS server with at least 2x CPU cores and 2 GB Ram
*  Ubuntu 18.04 LTS or 20.04 LTS
*  Create an A DNS record at your hosting provider for your explorer URL. (i.e. explorer.yourdomain.com)

### Requires

*  node.js >= 8.17.0 (12.14.0 is advised for updated dependencies)
*  mongodb 4.2.x
*  *coind

### Preparing the server

###### Install the latest updates
```bash
sudo apt update && sudo apt -y upgrade
```

###### When all updates are installed, reboot the server
```bash
sudo reboot
```

###### Install needed depencies and remove all unused packages
```bash
sudo apt-get install build-essential libtool autotools-dev automake pkg-config libssl-dev libevent-dev bsdmainutils python3 libboost-system-dev libboost-filesystem-dev libboost-chrono-dev libboost-test-dev libboost-thread-dev libboost-all-dev libboost-program-options-dev
```
```bash
sudo apt-get install libminiupnpc-dev libzmq3-dev libprotobuf-dev protobuf-compiler unzip software-properties-common libkrb5-dev git nginx gnupg nano screen
```
```bash
sudo apt -y autoremove --purge
```

###### When it’s done, reboot the server
```bash
sudo reboot
```

### Installation

#### Install Berkeley DB
```bash
sudo add-apt-repository ppa:bitcoin/bitcoin
sudo apt-get update
sudo apt-get install libdb4.8-dev libdb4.8++-dev
```

#### Install the coin daemon
Install the daemon reboot safe and add the parameters in the config file. Use no special characters for the username and password! Only the local host have rpc access, so the username and password mustn’t be very difficult and long. Start and sync the blockchain.
```
rpcuser=rpcuser
rpcpassword=rpcpassword
rpcallowip=127.0.0.1
listen=1
server=1
txindex=1
daemon=1
```

#### Install the Mongo Database Server
We are using the MongoDB version 4.2
```bash
wget -qO - https://www.mongodb.org/static/pgp/server-4.2.asc | sudo apt-key add -
```
```bash
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu bionic/mongodb-org/4.2 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-4.2.list && sudo apt update
```
```bash
sudo apt install -y mongodb-org
```
```bash
echo "mongodb-org hold" | sudo dpkg --set-selections
echo "mongodb-org-server hold" | sudo dpkg --set-selections
echo "mongodb-org-shell hold" | sudo dpkg --set-selections
echo "mongodb-org-mongos hold" | sudo dpkg --set-selections
echo "mongodb-org-tools hold" | sudo dpkg --set-selections
```

###### Configure and run the MongoDB
```bash
sudo systemctl daemon-reload && sudo systemctl start mongod && sudo systemctl enable mongod && sudo systemctl status mongod
```

###### Create the explorer database
```bash
mongo
use explorerdb
db.createUser( { user: "Your-DB-User", pwd: "Your-DB-Password", roles: [ "readWrite" ] } )
exit
```

*Note: If you're using mongo shell 4.2.x, use the following to create your user:
```
db.addUser( { user: "Your-DB-User", pwd: "Your-DB-Password", roles: [ "readWrite"] })
```

#### Install node.js and npm
```bash
sudo apt update && sudo apt -y install nodejs npm
```

#### Install the NEOS Explorer

###### Get the source
```bash
git clone https://github.com/Panther107th/NEOS-Explorer.git explorer
```

###### Install node modules
```bash
cd explorer && npm install --production
```

###### Configure
```bash
cp ./settings.json.template ./settings.json
```

*Make required changes in settings.json*

###### Start Explorer
```bash
screen -dmS explorer npm start
```

*note: mongod must be running to start the explorer*

As of version 1.4.0 the explorer defaults to cluster mode, forking an instance of its process to each cpu core. This results in increased performance and stability. Load balancing gets automatically taken care of and any instances that for some reason die, will be restarted automatically. For testing/development (or if you just wish to) a single instance can be launched with
```
node --stack-size=10000 bin/instance
```

**To attach to the screen you can use**
```bash
screen -r explorer
```

**To leave the screen you can use**
```
CTRL + A + D
```

**To stop the cluster you can use**
```bash
npm stop
```

The databases are created now. We will create indexes to speed up the db.
Stop the application
```bash
npm stop
```
```bash
mongo
use explorerdb
```
```bash
db.txes.createIndex({total: 1})
db.txes.createIndex({total: -1})
db.txes.createIndex({blockindex: 1})
db.txes.createIndex({blockindex: -1})
db.addresstxes.createIndex({a_id: 1, blockindex: -1})
exit
```

Start the application
```bash
npm start
```

Open a web browser and check the frontend. http://explorer.yourdomain.com:3001

### Nginx reverse proxy for security to run the app under port 80/443

Open the config file for nginx
```bash
sudo nano /etc/nginx/nginx.conf
```

Paste the config into the and change the server name with your URL.
```
server {
    listen 80;
    server_name explorer.yourdomain.com;

    location / {
        proxy_set_header   X-Forwarded-For $remote_addr;
        proxy_set_header   Host $http_host;
        proxy_pass         "http://127.0.0.1:3001";
    }
}
```

start the nginx reverse proxy.
```bash
sudo systemctl start nginx
```

Check: http://explorer.yourdomain.com

###### Secure your server

We will close all ports, except the ssh, http and https port.
```bash
ufw limit 22/tcp comment "Limit SSH "
ufw allow 22/tcp comment "SSH"
ufw allow http comment "HTTP"
ufw allow https comment "HTTPS"
```

###### Create a https certificate with Let’s Encrypt

Here an example to create a https certificate with Let’s Encrypt to have a secure browser session.

Add the certbot package repo. Press Enter to accept the key.
```bash
sudo add-apt-repository ppa:certbot/certbot
```
```bash
sudo apt install python-certbot-nginx
```

Obtaining an SSL Certificate. Replace the URL with yours.
```bash
sudo certbot --nginx -d explorer.yourdomain.com -d www.explorer.yourdomain.com
```

If that’s successful, certbot will ask how you’d like to configure your HTTPS settings.
```
Output
Please choose whether or not to redirect HTTP traffic to HTTPS, removing HTTP access.
-------------------------------------------------------------------------------
1: No redirect - Make no further changes to the webserver configuration.
2: Redirect - Make all requests redirect to secure HTTPS access. Choose this for
new sites, or if you're confident your site works on HTTPS. You can undo this
change by editing your web server's configuration.
-------------------------------------------------------------------------------
Select the appropriate number [1-2] then [enter] (press 'c' to cancel):
```

Choose here the option “2”.

Verifying Certbot Auto-Renewal
```bash
sudo certbot renew --dry-run
```

Finally you can configure the design and the correct API links. Switch trough the templates and choose what you like, replace the logo and the favicon.
To set the API links correct, choose a block and fill the settings.conf with the relevant values from your chain.

Create in /var/web/html a file “robots.txt” to prevent that the web crawlers index the explorer into public search engines.
```
User-agent: *
Disallow: /address
Disallow: /api
Disallow: /transaction
```

### Syncing databases with the blockchain

sync.js (located in scripts/) is used for updating the local databases. This script must be called from the explorers root directory.
```
Usage: node scripts/sync.js [database] [mode]

database: (required)
index [mode] Main index: coin info/stats, transactions & addresses
market       Market data: summaries, orderbooks, trade history & chartdata

mode: (required for index database only)
update       Updates index from last sync to current block
check        checks index for (and adds) any missing transactions/addresses
reindex      Clears index then resyncs from genesis to current block

notes:
* 'current block' is the latest created block when script is executed.
* The market database only supports (& defaults to) reindex mode.
* If check mode finds missing data(ignoring new data since last sync),
  index_timeout in settings.json is set too low.
```

*It is recommended to have this script launched via a cronjob at 1+ min intervals.*

**crontab**

*Example crontab; update index every minute and market data every 2 minutes*
```
*/1 * * * * cd /path/to/explorer && /usr/bin/nodejs scripts/sync.js index update > /dev/null 2>&1
*/5 * * * * cd /path/to/explorer && /usr/bin/nodejs scripts/peers.js > /dev/null 2>&1
```

### Clean the MongoDB

You can drop the mongo db with the following commands. It’s required it the db is corrupt.
```bash
mongo  
use explorerdb
```
```bash
db.addresses.remove({})
db.addresses.drop()
db.coinstats.remove({})
db.coinstats.drop()
db.markets.remove({})
db.markets.drop()
db.peers.remove({})
db.peers.drop()
db.richlists.remove({})
db.richlists.drop()
db.txes.remove({})
db.txes.drop()
exit
```

### Wallet
NEOS Explorer is intended to be generic so it can be used with any wallet following the usual standards. The wallet must be running with atleast the following flags
```
-daemon -txindex
```

### Donate   
```
BTC: 1FRWSExGkQRu8Zro9Cd5GfKbFKFtYDd2oR
ETH: 0x7B3E6DfF678aaF6eE4116FdE10a58397297a34a8
LTC: LgEm1NWhcT5dwQUwwww8vCHYKHjtPdPM55
```

### Known Issues

**script is already running.**

If you receive this message when launching the sync script either a) a sync is currently in progress, or b) a previous sync was killed before it completed. If you are certian a sync is not in progress remove the index.pid from the tmp folder in the explorer root directory.
```
rm tmp/index.pid
```

**exceeding stack size**
```
RangeError: Maximum call stack size exceeded
```

Nodes default stack size may be too small to index addresses with many tx's. If you experience the above error while running sync.js the stack size needs to be increased.

To determine the default setting run
```
node --v8-options | grep -B0 -A1 stack_size
```

To run sync.js with a larger stack size launch with
```
node --stack-size=[SIZE] scripts/sync.js index update
```

Where [SIZE] is an integer higher than the default.

*note: SIZE will depend on which blockchain you are using, you may need to play around a bit to find an optimal setting*

### License

Copyright (c) 2015, Iquidus Technology  
Copyright (c) 2015, Luke Williams
Copyright (c) 2020, NEOS Developers
All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:

* Redistributions of source code must retain the above copyright notice, this
  list of conditions and the following disclaimer.

* Redistributions in binary form must reproduce the above copyright notice,
  this list of conditions and the following disclaimer in the documentation
  and/or other materials provided with the distribution.

* Neither the name of Iquidus Technology nor the names of its
  contributors may be used to endorse or promote products derived from
  this software without specific prior written permission.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE
FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER
CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY,
OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
