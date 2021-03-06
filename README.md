# Roger-Skyline-1
Configuring a secure web server :) 


# Network and Security Part

### You must create a non-root user to connect to the machine and work.
In my case, I created a non-root user while setting up the virtual machine. All I have to do is input the username and password.

### Use sudo, with this user, to be able to perform operation requiring special rights.
To achive this, first install sudo. In order to install sudo, One must be in root user:
```
$ su || type root credentials upon vm startup
$ apt-get update -y && apt-get upgrade -y
$ apt-get install sudo vim -y
```

I prefer to use vim but you can use nano which is installed in the vm by default.

Next the `/etc/sudoers` file needs to be updated. The username you created must be given sudo rights.
In root user:
```
$ nano /etc/sudoers
```
Notice that I used nano. In this case nano will be the easiest because if I wanted to use vim I would have to change the read/write permissions of the file.

By default, on line 20 You will see that the root user has access to all. You will want to add your user to this file.
In my case:

```
# User privilege specification
root  ALL=(ALL:ALL) ALL
nunezcode ALL=(ALL:ALL) ALL
```
![sudoers](images/Sudoers.png)

Save the file and your new user now has sudo rights.

### We don’t want you to use the DHCP service of your machine. You’ve got to configure it to have a static IP and a Netmask in \30.

***1.*** First Things First, Open your VirtualBoxManager, Click on your current VM. Then click settings->Network-> On Adapter 1 change default NAT to Bridged Adapter.Then Save by clicking OK.

***2.*** Whilst on your sudo user: 
```
$ su - nunezcode (your user's username)
```
Type `ip addr` to check your changes. In my case, the name of my Bridged Adapter is `enp0s3` and just a couple lines under that, You will see its current ip address and its port.
![BridgeAdapter](images/BridgeAdapter.png)
Next We will configure its ip address by going into the interfaces file:

```
$ sudo vim /etc/network/interfaces
```

On line: 10 you should see the following:
![DefaultNetworkInterface](images/DefaultNetworkInterface.png)

Change the following lines with this:

```
auto enp0s3
```

The end result will look like this:
![UpdatedNetworkInterface](images/UpdatedNetworkInterface.png)

By doing this, You are telling the operating system that your configurations for the network `enp0s3` will be found inside the interfaces folder called `interfaces.d/`.

***4.*** Before adding the enp0s3 configurations, You will need to find the default gateway of host (Your lab computer). To get the gateway you need to run the following command on your host terminal:

```
$ netstat -rn
```

![DefaultGateway](images/Gateway.png)

In my case my default gateway is 10.113.254.254 This will be vital to hours of debugging trying to find out why you can't establish connection to the internet and download packages for your virtual machine. So make sure you do this.


***3.*** Next create the file for your enp0s3 configurations inside the interfaces.d folder:

```
$ sudo vim /etc/network/interfaces.d/enp0s3
```

The file should look similar to this depending on the gateway of your host machine:

![UpdatedStaticNetworkInterface](images/NetworkInterfaceStatic.png)

***4.*** After updating your enp0s3 you need to restart your network service in order for you changed to take effect: 

```
$ sudo service networking restart && ip addr 
```

![UpdatedRestart](images/UpdatedNetworkInterfaceRestart.png)

As you can see on the image, The ip address of enp0s3 has changed to the one you put in your interface file. In my case 10.113.1.142. Also, The state of my network adapter seems to be down for some reason, If this happens to you, all you have to do is run the command: 

```
sudo ifup enp0s3
```

### You have to change the default port of the SSH service by the one of your choice. SSH access HAS TO be done with publickeys. SSH root access SHOULD NOT be allowed directly, but with a user who can be root.

***1.*** You will need to change the port of the ssh service. To change the default port you will have to update the `sshd_config` file:

```
$ sudo vim /etc/ssh/sshd_config
```

Change the following:

Port 50550
PermitRootLogin no
PubkeyAuthentication yes

I chose port 50550. You can set yours to what ever you want as long as it doesn't confict with any other ports on your machine. PermitRootLogin will prevent someone from logging into the root user and doing things that you don't want done. The only way to access this machine will be from Users public keys.
>Port numbers are assigned in various ways, based on three ranges: System Ports (0-1023), User Ports (1024-49151), and the Dynamic and/or Private Ports (49152-65535);

***2.*** After updating your port number, restart the sshd service:
```
$ sudo service sshd restart
```

***3.*** After updating your sshd configurations, You will need to [create an SSH Public Key](https://www.cyberciti.biz/faq/ubuntu-18-04-setup-ssh-public-key-authentication/).

On your host terminal (Lab computer) generate an ssh certificate:

```
$ ssh-keygen -t rsa
```
This command will generate 2 files id_rsa and id_rsa.pub. id_rsa should be kept private and never shared with anyone. The one we need to transfer to the server is id_rsa.pub to do this, We use the `ssh-copy-id` command.

```
ssh-copy-id -i id_rsa.pub nunezcode@10.113.1.142 -p 50550
```

Notice that I input my user's `username` followed by the `static ip address` that we gave our server followed by the `-p` flag (which stands for port) and then we specify the `port we selected`.

This will automatically add the key you just created into  ~/.ssh/authorized_keys on the server.

***3.*** After you setup your public key, Go back to the virtual machine and edit the `/etc/ssh/sshd_config` one last time.
```
$ sudo vim /etc/ssh/sshd_config
```

Change PasswordAuthentification to "no".
```
PasswordAuthentification no
```

***4.*** Last but not least, restart your SSH service:
```
sudo service sshd restart
```

You should now be able to login to your server via ssh by running the following command from your host machine:

ssh username@ip -p port
```
ssh nunezcode@10.113.1.142 -p 50550
```

run the command `exit` on the host's terminal to exit the server command prompt.

### You have to set the rules of your firewall on your server only with the services used outside the VM.

```
$ sudo apt-get install ufw
$ sudo ufw status
$ sudo ufw enable
```

We will need to enable to following 3 services on our firewall:
  - SSH : `sudo ufw allow 50550/tcp` (Allow connection via ssh)
  - HTTP : `sudo ufw allow 80/tcp` (Allow HTTP)
  - HTTPS : `sudo ufw allow 443` (Allow HTTPS)
  - STATUS : `sudo ufw status` (Check the status of our firewall)
  
![FirewallStatus](images/FirewallStatus.png)

### You have to set a DOS (Denial Of Service Attack) protection on your open ports of your VM.

There is many ways to setup DOS Protection I used the article over at [Garron](https://www.garron.me/en/go2linux/fail2ban-protect-web-server-http-dos-attack.html). Here is a summary:

***1.*** Install Apache2, Fail2Ban and IpTables
```
$ sudo apt-get install  apache2 fail2ban iptables
```

***2.*** Next, edit the Fail2Ban configuration file:

```
$ sudo vim /etc/fail2ban/jail.conf
```

- Protect our open ssh port (50550): Look for `[sshd]` and edit its value with the following:

```
[sshd]
enable    = true
port      = ssh
logpath   = %(sshd_log)s
backend   = %(sshd_backend)s
maxentry  = 3
bantime   = 1200
```

- Protect our port 80 (HTTP protocol security): Look for `[http-get-dos]` if it doesn't exist just add it. In my case, It didn't. Put the following for its value:

```
[http-get-dos]
enabled   = true
port      = http, https
filter    = http-get-dos
logpath   = /var/log/apache2/access.log
maxretry  = 300
findtime  = 300
bantime   = 1200
action    = iptables[name=HTTP, port=http, protocol=tcp]
```

The following steps should overall look like this:
![ProtectedPorts](images/ProtectedPorts.png)

- Now create the filter. Create the file /etc/fail2ban/filter.d/http-get-dos.conf:

```
$ sudo vim /etc/fail2ban/filter.d/http-get-dos.conf
```

- Copy the text below in it:
```
[Definition]

failregex = ^ -.*GET

ignoreregex =
```

- Reload ufw and the fail2ban service: 
```
$ sudo ufw reload
$ sudo service fail2ban restart
```

- Run `sudo fail2ban-client status` to see the status of fail2ban jails:
![Fail2BanStatus](images/Fail2BanStatus.png)

### You have to set a protection against scans on your VM’s open ports.
The way I setup protection agains open port scans can be found over at [this](https://en-wiki.ikoula.com/en/To_protect_against_the_scan_of_ports_with_portsentry) link.

***1.*** Install Portsentry:

```
$ sudo apt-get install portsentry
```

***2.*** Configure Portsentry:

- Stop the portsentry service:

```
$ sudo /etc/init.d/portsentry stop
```

![StopPortSentry](images/StopPortSentry.png)

- "We will then implement the exceptions to not to block various IP addresses (at minimum your IP address)".

Edit the file /etc/portsentry/portsentry.ignore.static and add your ip address.

- We use portsentry in advanced mode for the TCP and UDP protocols. To do this, you must modify the file /etc/default/portsentry in order to have: 

```
$ sudo vim /etc/default/portsentry
```

Change to the following:

```
TCP_MODE="atcp"
UDP_MODE="audp"
```

- Portsentry needs to be a blockage. Activate it by changing BLOCK_UDP and BLOCK_TCP to 1:

Edit the file `/etc/portsentry/portsentry.conf` to match the following: 

```
BLOCK_UDP="1"
BLOCK_TCP="1"
```

- I want to block malicious attacks through iptables. Therefore comment on all lines of the configuration file that begin with KILL_ROUTE except the following: 

```
KILL_ROUTE="/sbin/iptables -I INPUT -s $TARGET$ -j DROP"
```

You can verify if you uncommented the right one by running the following command from inside /etc/portsentry : 

```
$ sudo cat portsentry.conf | grep KILL_ROUTE | grep -v "#"
```

- Relaunch portsentry service and it will begin to block the port scans:

```
$ sudo /etc/init.d/portsentry start
```

![PortSentryStart](images/PortSentryStart.png)

### Stop the services you don’t need for this project.

Install sysv-rc-conf:

```
$ sudo apt-get install sysv-rc-conf
```

Services are controlled with special shell scripts in `/etc/init.d`, Running this command will show all the services that are currently running : 

```
$ sudo service --status-all
```

To remove a service use the following command:

```
$ sudo update-rc.d SERVICE disable

$ sudo update-rc.d apparmor disable
$ sudo update-rc.d dbus disable
$ sudo update-rc.d kmod disable
```


### Create a script that updates all the sources of package, then your packages and which logs the whole in a file named /var/log/update_script.log. Create a scheduled task for this script once a week at 4AM and every time the machine reboots.

***1.*** I created a directory to store my maintenance scripts:

```
$ mkdir Scripts && touch Scripts/updatePackages.sh
$ chmod a+x ~/Scripts/updatePackages.sh
```

***2.*** Add following commands inside of `Scripts/updatePackages.sh` :

```
sudo apt-get update -y >> /var/log/update_script.log
sudo apt-get upgrade -y >> /var/log/update_script.log
```

***3.*** Add the task to crontab

```
$ sudo crontab -e
```

- In the file that was prompted add the following lines:

```
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin

@reboot sudo ~/Scripts/updatePackages.sh
0 4 * * 6 sudo ~/Scripts/updatePackages.sh
```

Save the file and reboot to see the logs inside of `/var/log/update_script.log`

### Make a script to monitor changes of the /etc/crontab file and sends an email to root if it has been modified. Create a scheduled script task every day at midnight.

***1.*** Create a file named `monitorCron.sh` inside of the `~/Scripts` folder:

```
$ touch ~/Scripts/monitorCron.sh
$ chmod a+x ~/Scripts/monitorCron.sh
```

- Add the following to the `monitorCron.sh` file: 

```
#!/bin/bash

sudo touch /home/nunezcode/Scripts/cronMd5
sudo chmod 777 /home/nunezcode/Scripts/cronMd5
m1="$(md5sum '/etc/crontab' | awk '{print $1}')"
m2="$(cat '/home/nunezcode/Scripts/cronMd5')"
echo ${m1}
echo ${m2}

if [ "$m1" != "$m2"] ; then
	md5sum /etc/crontabs | awk '{print $1}' > /home/nunezcode/Scripts/cronMd5
	echo "Crontab was edited!" | mail -s "Cronfile was changed" root@debian.lan
fi

```

- Make sure cron service is enabled: 

```
$ sudo systemctl enable cron
```

- Follow the following tutorial to recieve mail [tutorial](https://www.cmsimike.com/blog/2011/10/30/setting-up-local-mail-delivery-on-ubuntu-with-postfix-and-mutt/"

### Web Part

***1.*** Generate SSL self-signed key and certificate:

```
$sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/apache-selfsigned.key -out /etc/ssl/certs/apache-selfsigned.crt

Country name: UA
State or Province Name: ENTER
Locality Name: ENTER
Organization Name: ENTER
Organizational Unit Name: ENTER
Common Name: 10.113.1.142 (VM IP address)
Email Address: root@debian.lan
```

***2.*** Create the file `/etc/apache2/conf-available/ssl-params.conf` and add the following in it:

```
SSLCipherSuite EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH
SSLProtocol All -SSLv2 -SSLv3
SSLHonorCipherOrder On

Header always set X-Frame-Options DENY
Header always set X-Content-Type-Options nosniff

SSLCompression off
SSLSessionTickets Off
SSLUseStapling on
SSLStaplingCache "shmcb:logs/stapling-cache(150000)"
```






  



























