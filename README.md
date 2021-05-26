# CS:GO Dedicated Server

## Setting up the VM instance (GCP)

Assuming a Project is created on Google Cloud Platform (https://console.cloud.google.com/)

Go to `Compute Engine > VM Instances > Create Instance`.

* Name: a meaningful name
* Region: best region for your server (e.g: `southamerica-east1` for Brazilian servers)
* Zone: doesn't really matter here
* Machine Configuration: a suitable instance type. Smallest ones are under N1 sries.
* Boot disk: Ubuntu 20.04 LTS
* Boot disk type: Standard persistence disk (cheaper) -> Increase boot size to at least 50GB

Within the sidebar menu, go to `Networking > VPC Network > External IP Address` (or use search bar looking for `VPC`).

* Easiest way is clicking on `Ephemeral` under the `Type` column and change to `Static`. This will bind the existing address to an external one.
* Another option is creating a new Static Address and manually attaching to the instance.


## Connecting to the VM instance (GCP)

There are many options for this, but two good ones are:
* Use the `connect` option on the GCP Web console interface (right side). It's an embedded Web SSH console.
* Set up a SSH Key under `Compute > Compute Engine > Metadata > SSH Keys tab` to connect remotely from your local
    * Create local SSH Key, e.g: `ssh-keygen -t rsa -f ~/.ssh/keyname -C username` (username = GCP username, e.g: vitorbimranda)
    * Copy the content from the generated `~/.ssh/keyname.pub` public key and add it through the GCP SSH Keys page.


## Setting up the CS:GO Dedicated Server 

*This is a compiled version from https://developer.valvesoftware.com/wiki/Counter-Strike:_Global_Offensive_Dedicated_Servers*

### If using 64bit machines (likely)

```
sudo add-apt-repository multiverse
sudo dpkg --add-architecture i386
sudo apt update
sudo apt install lib32gcc1 -y
```

### Create 'steam' user and install SteamCMD

```
# ps: default vm instance user has sudo activated
sudo su

# this makes the steamcmd installer skip the license agreement
echo steamcmd steam/question select "I AGREE" | sudo debconf-set-selections
echo steamcmd steam/license note "" | sudo debconf-set-selections

# creates the 'steam' user to make operations isolated
useradd -m steam
apt install steamcmd
```

### Install CS:GO Dedicated Server


```
sudo su - steam
STEAM_DIR=/home/steam
cd $STEAM_DIR

# sets up the script that will be executed using steamcmd - this way you can run everything at once
# IMPORTANT: app_update can be unstable sometimes. If it fails, have to manually try it again.

echo "login anonymous" >> $STEAM_DIR/steamcmd_csgo.txt
echo "force_install_dir ./cs_go/" >> $STEAM_DIR/steamcmd_csgo.txt
echo "app_update 740 validate" >> $STEAM_DIR/steamcmd_csgo.txt
echo "quit" >> $STEAM_DIR/steamcmd_csgo.txt

# run the script
steamcmd +runscript $STEAM_DIR/steamcmd_csgo.txt

# in case it fails, put something to automatically retry it, e.g:
while true; do steamcmd +runscript $STEAM_DIR/steamcmd_csgo.txt; done

```

### Create the Game Server Login Token
This is necessary to make our server available to the internet. 

- Go to http://steamcommunity.com/dev/managegameservers
- On AppId use `730`
- Second field is for a custom identification only, e.g "myserver"

Save the generated session key.

### Configure the Linux daemon service

This makes stop/starting the service easier. Change `YOUR_GSLT_KEY_HERE` to the key created above on GLST.

```
sudo su
cat <<EOT >> /etc/systemd/system/csgo.service
[Unit]
Description=csgo

[Service]
ExecStart=/home/steam/.steam/steamcmd/cs_go/srcds_run -game csgo -insecure -debug -condebug -console -usercon +game_type 0 +game_mode 0 +mapgroup mg_active +map de_dust2 +sv_setsteamaccount YOUR_GSLT_KEY_HERE -net_port_try 1
Restart=on-failure
User=steam
Environment=PATH=/usr/bin:/usr/local/bin
WorkingDirectory=/home/steam/.steam/steamcmd/cs_go

[Install]
WantedBy=multi-user.target
EOT

# Ensure it's updated on systemctl
systemctl daemon-reload
```

## Updating the CS:GO Server

Eventually updates need to be applied to the server. If that's the case, you can set up something like this on crontab, in a sh file (e.g: updatecsgo.sh):

```
#!/bin/bash
set -x
sudo service csgo stop
sleep 5
sudo su - steam -c 'steamcmd +runscript /home/steam/steamcmd_csgo.txt'
sudo service csgo start
```

Then this can be added to crontab (`crontab -e`), e.g: `0 4 * * * /home/vitorbmiranda/updatecsgo.sh > /var/log/csgo/updatecsgo.log`

## Installing Plugins

We should generally want Metamod and Sourcemod plugins - and then other extra extensions such as RankMe or Retake.

**Important**: careful with file permissions. Always ensure that everything belongs to `steam` user, e.g: `sudo chown -R steam:steam /home/steam/.steam/steamcmd/cs_go/csgo`. If there are permission issues the plugins may not be loaded and error messages are not clear enough.

### Installing Metamod

* Follow instructions on https://wiki.alliedmods.net/Installing_Metamod:Source. It's basically downloading the zip or tar file and unzipping it to the addons dir. A `cp -r` on the directory should make it get merged with existing one. 

For a reference, content of `metamod.vdf` file on a working server was:

```
"Plugin"
{
	"file"	"../csgo/addons/metamod/bin/server"
}
```

### Installing Sourcemod

* Follow the Sourcemod part here: https://wiki.alliedmods.net/Installing_SourceMod_(simple)
* It's as easy as unpackaging the file into csgo root folder. It contains a `cfg` and an `addons` folder which should be merged into existing csgo root folder (again with `cp -r`). 

### Installing Sourcemod plugins

To install sourcemod plugins it's a matter of extracting the content into the `sourcemod` folder, such as `/home/steam/.steam/steamcmd/cs_go/csgo/addons/sourcemod`, unless the plugin documentation says otherwise. Again, need to ensure the files belong to `steam` user.

* https://github.com/rogeraabbccdd/Kento-Rankme
* https://github.com/splewis/csgo-retakes
* https://github.com/splewis/csgo-pug-setup
