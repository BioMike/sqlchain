### INSTALLATION GUIDE

At this time sqlChain is only tested on Linux servers. It may work on other platforms but I don't have them to test and probably won't put effort into that in the near future. 

There is a new sqlchain-init script that handles most of the configuration and DB init details. 

Tested on a clean Ubuntu 14.04 Amazon EC2 instance (ami-feed08e8). I used a spot m4.large instance which costs only 1.4 cents/hour to test (plus data transfer out, which seems to be pretty substanial, and maybe the main cost). 

### Getting Started - Step by Step

First, you need Bitcoin Core and some standard Ubuntu packages for MySQL and Python.

```
sudo apt-get install software-properties-common python-software-properties libev-dev libevent-dev   # may not need but won't hurt
sudo add-apt-repository ppa:bitcoin/bitcoin
sudo apt-get update
sudo apt-get install bitcoind mysql-server libmysqlclient-dev python-pip python-dev
```

Then you can install sqlChain from PyPi the easy way, includes dependencies and demo API web pages.

    sudo pip install sqlchain

That creates binaries in /usr/local/bin and puts python stuff where it should normally go.

The easy way to create the DB and configure and co-ordinate both bitcoin core and sqlchain daemons:

    sudo sqlchain-init
    
Answer the questions and it will create a user, mysql db and config files with correct permissions in locations you indicate. There are defaults for everything so you can get by with hitting 'Enter' the whole way thru. It also can be run again to change options but does not remove old files. It won't clobber the database either - you have to do that manually. It will create some demo api/web server files which you can build upon.

Finally, try starting the daemons, one at a time at first (this uses the upstart init process, which also starts them at boot).

```
sudo start bitcoin
sudo start sqlchain
sudo start sqlchain-api
```

If the process doesn't seem to start you can check in /var/log/upstart/ for logs upstart makes when something goes wrong. If a process starts but has other issues you can monitor them in their log files, by default directories as below,

```
/var/data/bitcoin/debug.log
/var/data/sqlchain/daemon.log
/var/data/sqlchain/api.log
```

You can add your normal user to the sqlchain/bitcoin group (by default "btc"),

    sudo adduser <myuser> btc
    
That allows you to use the config files easily, such as when using bitcoin-cli. It's also useful to have some aliases for common tasks. You can throw these in your .bashrc so they are present on login.

```
alias btc='bitcoin-cli -conf=/etc/bitcoin/bitcoin.conf'
alias sqdlog='sudo less /var/data/sqlchain/daemon.log'
```

### Updating

If you want to update sqlchain as I implement or fix stuff,

    sudo pip install --upgrade sqlchain
    
should do the trick. Bitcoin will get updated by the Ubuntu PPA package system (unless you do a custom build for manual pruning).

### Pruning Mode

If you select "pruning" mode in the sqlchain-init questions then it will send rpc calls to bitcoin to let it know when a block is processed. Bitcoin prunes in "block file units", each one being ~128MB. So when sqlchaind has completed all blocks in a given block file it is deleted. The pruning only works in manual mode and this is currently non-standard, though it appears it has been committed for bitcoin version 0.14.0

There is a nice pull request (#7871) in the bitcoin github to enable this but you would have to clone the repo, merge the pull request and build a custom binary. That's not hard to do and I can add a similar step-by-step if anyone requests it. It seems a custom build installs into /usr/local/bin instead of /usr/bin - so that overrides the normal bitcoind allowing you to use a full path to select which to start (change conf file to match).

### Other Details

By default the API server (sqlchain-api) listens on localhost:8085 but of course a simple edit of sqlchain-api.cfg allows changing that. For example, to listen on the normal public port, assuming root privileges (when started as root it will drop to chosen user after):

    "listen":"0.0.0.0:80",
    
You can also set an SSL certificate file to use if you want to serve https. I would suggest for public access it's better to use nginx as a reverse proxy. It's fairly easy to setup and better for several reasons, one being secure certificate handling. But, you can edit the sqlchain-api.cfg and add:

    "ssl":"path/to/full-chain-certificate-file",
    "key":"path/to/private-key-file",     (optional: don't set if you use a combined key+cert in above)
    
This could be a file with concatenated private key and certificate blocks, or separate files. It should have read-only permissions; due to how python handles ssl it needs to be readable by the running user.

A simple proxy config for nginx is below. You could have several api servers behind nginx and it can load balance across them. 

    server {
        listen 80;
        listen [::]:80;
        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        server_name example.com www.example.com;
    
        ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    
        location / {
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Host $http_host;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_pass http://127.0.0.1:8085;
        }
    }

It is also probably better for serving static content and only passing api calls to sqlchain-api, in which case use a blank value www cfg item to disable root files. eg.

    "www":"",

The sqlchain-api "dbinfo" cfg option sets whether db state queries are run and at what interval: -1 = never, 0 = at start only, >0 minute interval. The db info output is available as an API call. So the following will refresh info every 10 minutes.

    "dbinfo":10,

Any questions or suggestions - post issues on GitHub or comments on my blog or send a message through my web site.



