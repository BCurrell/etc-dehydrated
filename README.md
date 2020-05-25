# /etc/dehydrated

A collection of configs for a simple Dehydrated deployment

## Documentation

### Installing

This repository is strictly for the configs related to dehydrated SSL certificate generation, and does not have any control over the actual dehydrated package. As such, you will need to install dehydrated first, and then clone this repo into the correct location.

1. Install the prereqisite packages.
    - `apt-get update`
    - `apt-get install git dehydrated`
2. Remove the default configuration.
    - `cd /etc/dehydrated/`
    - `rm -rf *`
3. Clone this repository into the dehydrated configuration location, and initialise the submodules.
    - `git clone https://github.com/bcurrell/etc-dehydrated.git .`
    - `git submodule update --init --recursive`
4. Create the webserver location for http-01 challenges.
    - `mkdir -p /var/www/dehydrated/`
5. Create the dehydrated user and group.
    - `adduser --system --no-create-home --home /etc/dehydrated --shell /usr/bin/nologin --disabled-login --group dehydrated`
6. Set the directory permissions.
    - `chown -R dehydrated:dehydrated /etc/dehydrated/`
    - `chown -R dehydrated:www-data /var/www/dehydrated/`

The dehydrated user we created cannot be logged into to run commands with. Instead, if you want to run commands as dehydrated, use the following:

`sudo -u dehydrated -H <command>`

### Configuring

This repository comprises of a best-fit base dehydrated config, as well as any custom configuration we want to deploy. For the most part, none of these config files need changing once the repo is cloned / pulled to each system. However in the event that something needs adjusting for updates / debugging, the base config is fully commented and the other config files are documented on dehydrated's [GitHub page](https://github.com/dehydrated-io/dehydrated/tree/master/docs).

One thing that will need configuring after installation is the hooks. You will most likely need to create / edit a .env or config file within the hook directory for authentication of automated DNS updates. Check the README of each of the hooks for instructions on initialisation, configuration and running.

If you need to configure new domains or hooks, follow the below steps, and be sure to push any updates to Git so they can be loaded on all systems.

1. Add all changes to git local repository.
    - `git add --all`
2. Add a commit message describing the changes made.
    - `git commit -am "Added new domain and associated hook"`
3. Push to git remote repository.
    - `git push origin master`

#### Adding domains

There is 1 required and 1 optional file change required for adding a single or set of domains. This is documented on dehydrated's GitHub page as mentioned above, but these instructions are specific to our setup.

1. Required: Update /etc/dehydrated/domains.txt.
    - Add a new line of space-separated domain names that you wish to generate certificates for.
        - `example.com *.example.com`
    - A line consisting of just domains will take the first provided domain as the alias for the collection.
        - `example.com == example.com *.example.com`
    - A line ending in a greater than sign and string without spaces will use the provided string as the alias for the collection.
        - `wildcard_example.com == example.com *.example.com > wildcard_example.com`
    - The provided or determined alias will be used as the name of the domain-specific config file, as well as the name of the directory within /etc/dehydrated/certs.d/ where the certificates and keys will be placed.
2. Optional: Create domain-specific config file.
    - Create a file in /etc/dehydrated/domains.d/<domain_alias>
        - `nano /etc/dehydrated/domains.d/wildcard_example.com`
    - Add config variables to file that will override those set in /etc/dehydrated/config. Usually this is hook-specific variables.
        - `CHALLENGETYPE="dns-01"`
        - `HOOK="/etc/dehydrated/hooks.d/dns-hurricane-electric-example/hook.sh"`
        - `HOOK_CHAIN="yes"`

#### Adding hooks

Whilst it is possible to just create any directories and files you need within /etc/dehydrated/hooks.d/, the better way is to use a separate Git repository for the hook itself for easier reusability. We achieve this with submodules. You can have multiple submodules at different paths using the same Git repository, even using different branches and commits.

1. Access the hooks.d directory.
    - `cd /etc/dehydrated/hooks.d/`
2. Create and initialise the submodule.
    - `git submodule add <git-repository-url> <path>`
        - `Example: git submodule add https://github.com/bcurrell/dehydrated-dns-hurricane-electric dns-hurricane-electric-example`
    - `git submodule update --init --recursive`

### Running

Now that dehydrated is installed and configured, you need to run it to generate the certificates. This is done with a single command, and will take a few minutes to run.

`dehydrated -c`

This command should be run on a regular schedule to automatically renew the certificates when they are due to expire within the configured number of days. The recommendation is that the renewal period is set to 30 days, with a 7 day schedule running the above command.

The automated running of the dehydated command can be done with a number of methods, the most common being a crontab or systemd service and timer. We will be deploying the latter to better fit current standards.

1. Create dehydrated.service

```none
nano /etc/systemd/system/dehydrated.service

[Unit]
Description=Dehydrated certificate renewal service

[Service]
Type=oneshot
User=dehydrated
Group=dehydrated
ProtectSystem=strict
ProtectHome=yes
ReadWritePaths=/etc/dehydrated /var/www/dehydrated
PrivateTmp=yes
ExecStart=/usr/bin/dehydrated -c
ExecStartPost=+/bin/systemctl reload nginx.service
```

2. Test dehydrated.service

```none
systemctl daemon-reload
systemctl start dehydrated.service
# This will take a while to run, especially if you haven't run dehydrated -c manually yet
systemctl status -l dehydrated.service
```

3. Create dehydrated.timer

```none
nano /etc/systemd/system/dehydrated.timer

[Unit]
Description=Dehydrated certificate renewal timer

[Timer]
# Run service 1 hour after initial boot...
OnBootSec=1h
# ...and then every 1 week afterwards
OnUnitActiveSec=1w
Persistent=true

[Install]
WantedBy=timers.target
```

4. Start and enable dehydrated.timer

```none
systemctl daemon-reload
systemctl enable dehydrated.timer
systemctl start dehydrated.timer
systemctl status -l dehydrated.timer
```

### Updating

As this repository and its submodules are being cloned on the relevant systems, updating is as simple as running `git pull` in each of the modules. The quick version is below:

1. Access the dehydrated config directory
    - `cd /etc/dehydrated`
2. Pull the main changes
    - `git pull origin master`
3. Pull the submodule changes
    - `git submodule foreach --recursive git pull`
