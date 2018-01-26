# Securing your Raspberry Pi

The security of your Raspberry Pi is important. Gaps in security leave your Raspberry Pi open to hackers who can then use it without your permission.

What level of security you need depends on how you wish to use your Raspberry Pi. For example, if you are simply using your Raspberry Pi on your home network, behind a router with a firewall, then it is already quite secure by default.

However, if you wish to expose your Raspberry Pi directly to the internet, either with a direct connection (unlikely) or by letting certain protocols through your router firewall (e.g. SSH), then you need to make some basic security changes.

Even if you are hidden behind a firewall, it is sensible to take security seriously. This documentation will describe some ways of improving the security of your Raspberry Pi. Please note, though, that it is not exhaustive.

## Change your default password

The default username and password is used for every single Raspberry Pi running Raspbian. So, if you can get access to a Raspberry Pi, and these settings have not been changed, you have `root` access to that Raspberry Pi.

So the first thing to do is change the password. This can be done via the raspi-config application, or from the command line.

```
sudo raspi-config
```

Select option 2, and follow the instructions to change the password.

In fact, all raspi-config does is start up the command line `passwd` application, which you can do from the command line. Simply type in your new password and confirm it.

```
passwd
```

## Changing your username

You can, of course, make your Raspberry Pi even more secure by also changing your username. All Raspberry Pis come with the default username `pi`, so changing this will immediately make your Raspberry Pi more secure.

To add a new user with the same permissions as the `pi` user:

```
sudo useradd -m fred -G sudo
```

This adds a new user called `fred`, creates a home folder, and adds the user to the `sudo` group. You now need to set a password for the new user:

```
sudo passwd fred
```

Log out and log back with the new account details. Check your permissions are in place (i.e. you can sudo) by trying the following.

```
sudo visudo
```

The `visudo` command can only be run by an account with sudo privileges. If it runs successfully, then you can be sure that the new account is in the `sudo` group.

Once you have confirmed that the new account is working, you can delete the `pi` user. Please note, though, that with the current Raspbian distribution, there are some aspects that require the `pi` user to be present. If you are unsure whether you will be affected by this, then leave the `pi` user in place. Work is being done to reduce the dependency on the `pi` user.

To delete the `pi` user, type the following: 

```
sudo deluser pi
```

This command will delete the `pi` user but will leave the `home/pi` folder. If necessary, you can use the command below to remove the home folder for the `pi` user at the same time. Note the data in this folder will be permanently deleted, so make sure any required data is stored elsewhere.

```
sudo deluser -remove-home pi
```

## Make `sudo` require a password

Placing `sudo` in front of a command runs it as a superuser, and by default, that does not need a password. In general, this is not a problem. However, if your Pi is exposed to the internet and somehow becomes exploited (perhaps via a webpage exploit for example), the attacker will be able to change things that require superuser credential, unless you have set `sudo` to require a password. 

To force `sudo` to require a password, enter

```
sudo nano /etc/sudoers.d/010_pi-nopasswd
```

and change the `pi` entry (or whichever usernames have superuser rights) to

```
pi ALL=(ALL) PASSWD: ALL
```

Now save the file. 

## Ensure you have the latest security fixes

This can be as simple as ensuring your version of Raspbian is up-to-date, as an up-to-date distribution contains all the latest security fixes. Full instructions can be found [here](../raspbian/updating.md).

If you are using SSH to connect to your Raspberry Pi, it can be worthwhile to add a cron job that specifically updates the ssh-server. The following command, perhaps as a daily cron job, will ensure you have the latest SSH security fixes promptly, independent of your normal update process. More information on setting up cron can be found [here](../linux/usage/cron.md)

```
apt-get install openssh-server
```

## Improving SSH security

SSH is a common way of accessing a Raspberry Pi remotely. By default, logging in with SSH requires a username/password pair, and there are ways to make this more secure. An even more secure method is to use key based authentication.

### Improving username/password security

The most important thing to do is ensure you have a very robust password. If your Raspberry Pi is exposed to the internet, the password needs to be very secure. This will help to avoid dictionary attacks or the like.

You can also **allow** or **deny** specific users by altering the `sshd` configuration.

```
sudo nano /etc/ssh/sshd_config
```

Add, edit, or append to the end of the file the following line, which contains the usernames you wish to allow to log in:

```
AllowUsers edward andrew charles anne
```

You can also use DenyUsers to specifically stop some usernames from logging in:

```
DenyUsers harry william
```

After the change you will need to restart the `sshd` service using `sudo systemctl restart ssh` or reboot so the changes take effect.

### Using key-based authentication.

Key pairs are two cryptographically secure keys. One is private, and one is public. They can be used to authenticate a client to an SSH server (in this case the Raspberry Pi). 

The client generates two keys, which are cryptographically linked to each other. The private key should never be released, but the public key can be freely shared. The SSH server takes a copy of the public key, and, when a link is requested, uses this key to send the client a challenge message, which the client will encrypt using the private key. If the server can use the public key to decrypt this message back to the original challenge message, then the identity of the client can be confirmed. 

Generating a key pair in Linux is done using the `ssh-keygen` command on the **client**; the keys are stored by default in the `.ssh` folder in the user's home directly. The private key will be called `id_rsa` and the associated public key will be called `id_rsa.pub`. The key will be 2048 bits long: breaking the encryption on a key of that length would take an extremely long time, so is very secure. You can make longer keys if the situation demands it. Note that you should only do the generation process once: if repeated, it will overwrite any previous generated keys. Anything relying on those old keys will need to be updated to the new keys. 

You will be prompted for a passphrase during key generation: this is an extra level of security. For the moment, leave this blank.

The public key now needs to be moved on to the server. This can be done by email, or cut and paste, or file copying. Once on the server it needs to be added to the SSH systems authorised keys. It should be emphasised that the `id_rsa` file is the private key and SHOULD NOT LEAVE THE CLIENT, whilst the public key file is `id_rsa.pub`.

Add the new public key to the authorisation file as follows:

```
cat id_rsa.pub >> ~/.ssh/authorized_keys
```

Alternatively, you can edit the file `sudo nano ~/.ssh/authorized_keys` and copy/paste the key in. It is perfectly acceptable to have multiple entries in the authorized_keys file, so SSH can support multiple clients.

Note that the authorized_keys file needs the correct permissions to be read correctly by the `ssh` system.

```
sudo chmod 644 ~/.ssh/authorized_keys
```

Finally, we need to disable password logins, so that all authentication is done by the key pairs.

```
sudo nano /etc/ssh/sshd_config
```

There are three lines that need to be changed to `no`, if they are not set that way already:

```
ChallengeResponseAuthentication no
PasswordAuthentication no
UsePAM no
```

Save the file and either restart the ssh system with `sudo service ssh reload` or reboot.

## Install a firewall

There are many firewall solutions available for Linux. Most use the underlying [iptables](http://www.netfilter.org/projects/iptables/index.html) project to provide packet filtering. This project sits over the Linux netfiltering system. `iptables` is installed by default on Raspbian, but is not set up. Setting it up can be a complicated task, and one project that provides a simpler interface than `iptables` is [ufw](https://www.linux.com/learn/introduction-uncomplicated-firewall-ufw), which stands for 'Uncomplicated Fire Wall'. This is the default firewall tool in Ubuntu, and can be easily installed on your Raspberry Pi: 

```
sudo apt-get install ufw
```

`ufw` is a fairly straightforward command line tool, although there are some GUIs available for it. This document will describe a few of the basic command line options. Note that `ufw` needs to be run with superuser privileges, so all commands are preceded with `sudo`. It is also possible to use the option `--dry-run` any `ufw` commands, which indicates the results of the command without actually making any changes.

To enable the firewall, which will also ensure it starts up on boot, use:

```
sudo ufw enable
```

To disable the firewall, and disable start up on boot, use: 

```
sudo ufw disable
```

Allow a particular port to have access (we have used port 22 in our example):

```
sudo ufw allow 22
```

Denying access on a port is also very simple (again, we have used port 22 as an example):

```
sudo ufw deny 22
```

You can also specify which service you are allowing or denying on a port. In this example, we are denying tcp on port 22:

```
sudo ufw deny 22/tcp
```

You can specify the service even if you do not know which port it uses. This example allows the ssh service access through the firewall:

```
sudo ufw allow ssh
```

The status command lists all current settings for the firewall:

```
sudo ufw status
```

The rules can be quite complicated, allowing specific IP addresses to be blocked, specifying in which direction traffic is allowed, or limiting the number of attempts to connect, for example to help defeat a Denial of Service (DoS) attack. You can also specify the device rules are to be applied to (e.g. eth0, wlan0). Please refer to the `ufw` man page (`man ufw`) for full details, but here are some examples of more sophisticated commands.

Limit login attempts on ssh port using tcp: this denies connection if an IP address has attempted to connect six or more times in the last 30 seconds: 

```
sudo ufw limit ssh/tcp
```

Deny access to port 30 from IP address 192.168.2.1

```
sudo ufw deny from 192.168.2.1 port 30
```

## Installing fail2ban

If you are using your Raspberry Pi as some sort of server, for example an ```ssh``` or webserver, your firewall will have deliberate 'holes' in it to let the server traffic through. In these cases, [Fail2ban](http://www.fail2ban.org) can be useful. Fail2ban, written in Python, is a scanner that examines the log files produced by the Raspberry Pi, and checks them for suspicious activity. It catches things like multiple brute-force attempts to log in, and can inform any installed firewall to stop further login attempts from suspicious IP addresses. It saves you having to manually check log files for intrusion attempts and then update the firewall (via `iptables`) to prevent them.

Install Fail2ban using the following command:

```
sudo apt-get install fail2ban
```

Note that the version of Fail2ban in the repository (v0.8.13) does not support IPv6 networks. If you use IPv6, you will need to install version v0.10 or higher from source. Please see the [Fail2ban](http://www.fail2ban.org) website for more information on how to do this.

On installation, Fail2ban creates a folder `/etc/fail2ban` in which there is a configuration file called `jail.conf`. This needs to be copied to `jail.local` to enable it. Inside this configuration file are a set of default options, together with options for checking specific services for abnormalities. Do the following to examine/change the rules that are used for `ssh`:

```
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```

Look for the section on `[ssh]`. It will look something like this.

```
[ssh]
enabled  = true
port     = ssh
filter   = sshd
logpath  = /var/log/auth.log
maxretry = 6
```

As you can see, this section is named ssh, is enabled, examines the ssh port, filters using the `\etc\fail2ban\filters.d\sshd.conf` parameters, parses the /var/log/auth.log for malicious activity, and allows six retries before the detection threshold is reached. Checking the default section, we can see that the default banning action is:

```
# Default banning action (e.g. iptables, iptables-new,
# iptables-multiport, shorewall, etc) It is used to define
# action_* variables. Can be overridden globally or per
# section within jail.local file
banaction = iptables-multiport
```

`iptables-multiport` means that the Fail2ban system will run the `/etc/fail2ban/action.d/iptables-multiport.conf` file when the detection threshold is reached. There are a number of different action configuration files that can be used. Multiport bans all access on all ports.

If you want to permanently ban an IP address after three failed attempts, you can change the maxretry value in the `[ssh]` section, and set the bantime to a negative number:

```
[ssh]
enabled  = true
port     = ssh
filter   = sshd
logpath  = /var/log/auth.log
maxretry = 3 
bantime = -1
```

There is a good tutorial on some of the internals of Fail2ban [here](https://www.digitalocean.com/community/tutorials/how-fail2ban-works-to-protect-services-on-a-linux-server). 