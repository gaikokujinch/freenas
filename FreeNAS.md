FreeNAS
=======

ToC
--------

+ [Dependencies](#dependencies)

+ [Plex](#plex)
+ [PlexPy](#plexpy)
+ [Testing](#testing)
+ [Backups](#backups)
+ [Crashplan](#crashplan)
+ [Appendix](#appendix)

Notes
-----

+ **nas**: My zpool
+ **10.0.0.11**: My static FreeNAS ip
+ **192.168.1.3**: My static media jail ip
+ **192.168.1.4**: My static backup jail ip

Along with that be sure to keep paths consistent between your builds. It is easy to forget if you SSH'd into FreeNAS or into the jail.

Additionally, **do not copy/paste entire chunks of commands**. I often skip over different yes/no options for brevity in this guide. Read what the prompt says and feel free to drop a comment if answers seem ambiguous.


Setting up the jail
-------------------
**Create a jail using the FreeNAS web UI**
```
Jail name: media_jail
IPv4 address: 192.168.1.3/24
autostart: checked
type: pluginjail
VIMAGE: unchecked
vanilla: checked
sysctls: allow.raw_sockets=true,allow.sysvipc=true
```

**Ports and dependencies<a name="#dependencies">**
```
ssh root@10.0.0.11
jls
jexec 17 tcsh
passwd
portsnap fetch extract
portsnap fetch update
sysrc sshd_enable=YES
sysrc ftpd_enable=YES
cd /usr/ports/ports-mgmt/pkg/ && make deinstall
cd /usr/ports/ports-mgmt/pkg/ && make install clean
pkg2ng
cd /usr/ports/ports-mgmt/portmaster && make config-recursive install clean
cd /usr/ports/devel/git && make config-recursive install clean
cd /usr/ports/devel/py-cheetah && make config-recursive install clean
```

**Create a 'media' user and create media directory**
```
mkdir /mnt/media
adduser
Username   : media
Password   : <blank>
Full Name  : Media
Uid        : 1001
Class      : 
Groups     : media 
Home       : /home/media
Home Mode  : 
Shell      : /bin/tcsh
Locked     : no
id media
```

Create this user in your FreeNAS with the same uid and gid (typically 1001 if you haven't made a custom account yet).

Add mounts for media + crashplan backups inside the jail (/mnt/<ZFS>/media to /mnt/media).

Software
--------


<a name="plex"></a>
**Plex**

```
cd /usr/ports/multimedia/plexmediaserver && make config-recursive install clean
cd /usr/local && mkdir plexdata
chown -R media:media plexdata
sysrc plexmediaserver_enable=YES
sysrc plexmediaserver_user=media
sysrc plexmediaserver_group=media
```

There has been new security settings added and there was a problem with the plex scanner making a request to a local agent. To fix it you should add `192.168.1.0/24` to Settings -> Server -> Network -> Enabled Advanced -> "List of IP Address and networks that are allowed without auth" add `192.168.1.0/24`. After that, click refresh all and the scanner should be able to connect to the local agent.

<a name="plexpy"></a>
**PlexPy**

```
cd /usr/local && git clone https://github.com/JonnyWong16/plexpy.git
chown -R media:media plexpy
cp /usr/local/plexpy/init-scripts/init.freebsd /usr/local/etc/rc.d/plexpy
chmod +x /usr/local/etc/rc.d/plexpy
sysrc plexpy_enable=YES
sysrc plexpy_user=media
sysrc plexpy_dir=/usr/local/plexpy
sysrc plexpy_port=8083
```



<a name="testing"></a>
Testing
-------
```
service sshd start
service couchpotato start
service sickrage start
service headphones start
service sabnzbd start
service plexmediaserver start
service deluge start
service nginx start
service php-fpm start
service postgres start
service calibre start
service madsonic start
service plexpy start
```

Now check to make sure everything is running fine (<a href="http://192.168.1.3">192.168.1.3</a>). Then shut down the plugin server and start it up again. Everything should still be working fine.

<a name="Backups"></a>
Backups
-------

**Important files**
Sickbeard replacement: sickbeard.db and config.ini
Sabnzbd replacement: sabnzbd.ini
Couch potato replacement:
Plex: not worth it

**Moving over databases and config files from plugins**
Log into FreeNAS
```
cp /mnt/tetra/plugins_1/usr/pbi/sickbeard-amd64/data/config.ini /mnt/tetra/media_jail/usr/local/sickbeard
cp /mnt/tetra/plugins_1/usr/pbi/sickbeard-amd64/data/sickbeard.db /mnt/tetra/media_jail/usr/local/sickbeard
cp /mnt/tetra/plugins_1/usr/pbi/sabnzbdplus-amd64/sabnzbd/sabnzbd.ini /mnt/tetra/media_jail/usr/local/sabnzbd
cp /mnt/tetra/plugins_1/usr/pbi/headphones-amd64/data/config.ini /mnt/tetra/media_jail/usr/local/headphones/
cp /mnt/tetra/plugins_1/usr/pbi/headphones-amd64/data/headphones.db /mnt/tetra/media_jail/usr/local/headphones/
cp /mnt/tetra/plugins_1/usr/local/CouchPotatoServer/data/settings.conf /mnt/tetra/media_jail/usr/local/CouchPotatoServer/data
cp /mnt/tetra/plugins_1/usr/local/CouchPotatoServer/data/couchpotato.db /mnt/tetra/media_jail/usr/local/CouchPotatoServer/data
```

Make sure your settings move across the boundary. Daemons might not start up if ip's, filepaths, etc. are different.

<a name="crashplan"></a>
**Crashplan**

*Create a jail using the FreeNAS web UI*
```
Jail name: backup_jail
IPv4 address: 192.168.1./24
autostart: checked
type: portjail
VIMAGE: unchecked
vanilla: checked
```

Now add the directories you want to backup and where the backups should go.

**Ports and dependencies**
```
ssh root@10.0.0.11
jls
jexec 5 tcsh
passwd
portsnap fetch && portsnap extract && portsnap update
sysrc sshd_enable=YES
vi /etc/ssh/sshd_config
# add the following to the end of the file
Match User crasheva
    AllowTcpForwarding yes

adduser # backup with Uid 1002
```
Download the java runtime in order to please the CrashPlan overlords (or alternatively, modify the crashplan port scripts):
http://www.oracle.com/technetwork/java/javase/downloads/index.html
Scroll down to Java SE 8u111/112 and click JRE Download (third button). Accept the license 

```

mkdir /usr/ports/distfiles/
cd usr/ports/distfiles/

fetch http://download.oracle.com/otn-pub/java/jdk/8u112-b15/jre-8u112-linux-i586.tar.gz
fetch http://download.oracle.com/otn-pub/java/jdk/8u112-b15/jdk-8u112-linux-i586.tar.gz
fetch http://download.oracle.com/otn-pub/java/jdk/8u112-b15-demos/jdk-8u112-linux-i586-demos.tar.gz


```


This is just to appease the makefile, and instead we will use OpenJDK's JRE anyways. The following takes quite a while to install.

Install java and crashplan
```

jexec crasheva tcsh
cd /usr/ports/java/openjdk8-jre/ && make install clean BATCH=yes
# have all options at the beginning cd /usr/ports/java/openjdk8-jre/ && make config-recursive && make install clean


cd /usr/ports/sysutils/linux-crashplan/ && make config-recursive && make install clean

 BATCH=yes
sysrc crashplan_enable=YES
```
Now change the default Java binary path:

```
vi /usr/local/share/crashplan/install.vars
  JAVACOMMON=/usr/local/bin/java
```

Restart the jail. Now follow [this guide](http://support.code42.com/CrashPlan/Latest/Configuring/Configuring_A_Headless_Client) to modify your current crashplan install to work on the remote machine. You will need to create a port bridge over ssh, which you can do with the following command (before starting up crashplan locally):
```
ssh -L 4200:localhost:4243 backup@192.168.1.4
```
And grab the file /var/lib/crashplan/.ui_info from the server and bring it to your local host where you will run the crashplan desktop client.

Helpful codes
-------------
Mounting USB drive:
```
kldload fuse
mkdir /mnt/usb
ntfs-3g /dev/da1s1 /mnt/usb
ntfs-3g -o permissions /dev/da1s1 /mnt/usb
```

<a name="upgrading"></a>
Upgrading
=========
Upgrading can be a royal pain... but fear not. Typically you can just run a portmaster -ad, and if it says "conflict... blah blah" just run "pkg delete -f <stupid old package>" then re-run the portmaster command. Eventually everything should be updated! Many times the update process comes to a grinding halt because of dependency issues. You can kick off a single app to be updated similar below. You will also want to review /usr/ports/UPDATING if you run into trouble to see if a port has changed. There is usually a command to migrate a package such as a `portmaster -o oldpackage newpackage`.
```
less /usr/ports/UPDATING
portsnap fetch update
cd /usrports/ports-mgmt/pkg && make install clean
cd /usr/ports/ports-mgmt/portmaster && make install clean
pkg version -l '<'
portmaster -Rafd
portmaster -fd news/sabnzbdplus
```

Rsync files
```
&rsync --progress --stats --recursive --times --perms --links --dry-run /mnt/tetra /mnt/usb/tetra
nohup foo &

rsync -az -H --delete --numeric-ids --stats --progress -e ssh root@10.0.0.11:/mnt/tetra/family /media/jacob/usb/tetra
rsync -az -H --delete --numeric-ids --stats --progress -e ssh root@10.0.0.11:/mnt/tetra/media_jail/usr/local/sickbeard/data/config.ini /media/jacob/usb/tetra/backup

cp 
```

Copy server and daemon config files and databases
```
mkdir /mnt/tetra/backup/server_configs
cd /mnt/tetra/backup/server_configs
rsync -aqz /mnt/tetra/media_jail/usr/local/sickbeard/config.ini /mnt/tetra/media_jail/usr/local/sickbeard/sickbeard.db sickbeard/
rsync -aqz /mnt/tetra/media_jail/usr/local/sabnzbd/sabnzbd.ini sabnzbd/
rsync -aqz /mnt/tetra/media_jail/usr/local/headphones/config.ini /mnt/tetra/media_jail/usr/local/headphones/headphones.db headphones/
rsync -aqz /mnt/tetra/media_jail/usr/local/CouchPotatoServer/data/settings.conf /mnt/tetra/media_jail/usr/local/CouchPotatoServer/data/couchpotato.db couchpotato/
rsync -aqz /mnt/tetra/media_jail/usr/local/etc/nginx/nginx.conf /mnt/tetra/media_jail/usr/local/www/home/index.html nginx/
```

```
cd /mnt/tetra/backup/server_configs
rsync -aqz sabnzbd/sabnzbd.ini /mnt/tetra/media_jail/usr/local/sabnzbd/
rsync -aqz sickbeard/config.ini sickbeard/sickbeard.db /mnt/tetra/media_jail/usr/local/sickbeard/
rsync -aqz headphones/config.ini headphones/headphones.db /mnt/tetra/media_jail/usr/local/headphones/
rsync -aqz couchpotato/settings.conf couchpotato/couchpotato.db /mnt/tetra/media_jail/usr/local/CouchPotatoServer/data/

cd /usr/local && chmod -R media:media sabnzbd sickbeard headphones CouchPotatoServer
```

<a name="appendix"></a>
Appendix
=========
