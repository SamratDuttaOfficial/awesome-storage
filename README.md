### Awesome Storage - SSH and SCP using Ngrok, plus NAS
---

#### Creating the SSH server:

Startig SSH on boot can be enabled with a simple command: 
```
sudo systemctl enable ssh
```

Or you can use it once using `sudo systemctl start sshd`.

In some cases this strategy might not work. For those cases, try the following commands:
```
sudo update-rc.d ssh defaults
sudo systemctl enable ssh.socket
sudo systemctl enable ssh.service
```

You can check your `sshd.service` file using `sudo cat /etc/systemd/system/sshd.service`. My file looks like the following:
```
[Unit]
Description=OpenBSD Secure Shell server
Documentation=man:sshd(8) man:sshd_config(5)
After=network.target auditd.service
ConditionPathExists=!/etc/ssh/sshd_not_to_be_run

[Service]
EnvironmentFile=-/etc/default/ssh
ExecStartPre=/usr/sbin/sshd -t
ExecStart=/usr/sbin/sshd -D $SSHD_OPTS
ExecReload=/usr/sbin/sshd -t
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure
RestartPreventExitStatus=255
Type=notify
RuntimeDirectory=sshd
RuntimeDirectoryMode=0755

[Install]
WantedBy=multi-user.target
Alias=sshd.service
```

If all these don't work, try creating the sshd.service file yourself and enable it. Follow these steps to achieve that:

1. Create a new Systemd unit file for SSH: `sudo nano /etc/systemd/system/sshd.service`.
2. Add the following contents to the file:
    ```
    [Unit]
    Description=OpenSSH Server
    After=network.target
    
    [Service]
    ExecStart=/usr/sbin/sshd
    Restart=on-failure
    
    [Install]
    WantedBy=multi-user.target
    ```
3. Reload the Systemd daemon: `sudo systemctl daemon-reload`.
4. Enable the SSH service: `sudo systemctl enable sshd`.
5. Start the SSH service: `sudo systemctl start sshd`.

For more information: https://askubuntu.com/questions/892447/start-ssh-automatically-on-boot

#### Setting up Ngrok: 

To set up ngrok in a way that it always starts when the machine starts, follow the steps below. 

1. You can install ngrok using the following command: `snap install ngrok`.
2. Now that you have installed ngrok on your Linux device, link it to your ngrok account by using your authtoken: `ngrok authtoken NGROK_AUTHTOKEN`. Replace NGROK_AUTHTOKEN with your unique ngrok authtoken found in the ngrok dashboard (https://dashboard.ngrok.com/get-started/your-authtoken)
3. You can fire up an easy SSH access with ngrok: `ngrok tcp 22`. 
4. To make ngrok start and run the `ngrok tcp 22` command automatically whenever the system starts, we have to make a systemd service file. Create a systemd service file named ngrok.service with the following contents:
    ```
    [Unit]
    Description=ngrok
    After=network.target
    
    [Service]
    Type=simple
    ExecStart=/snap/bin/ngrok tcp 22
    RuntimeMaxSec=7100s
    Restart=always
    
    [Install]
    WantedBy=multi-user.target
    ```
    Here, `/snap/` is where ngrok has been installed (since we installed it using snap), and `/snap/bin/` is where the ngrok executable lie. Change it if you installed ngrok differently. The `RuntimeMaxSec` is set at 7100 seconds, which is roughly 118 minutes. Since ngrok's service automatically ends every 2 hours, we are restarting our service every 2 hours (almost). This way, we will always have an ngrok service running. Although every 2 hours you will be assigned a new address, but that you can check from ngrok's website time to time.  
5. To make the file, use the command: `sudo nano /etc/systemd/system/ngrok.service`. Write the contents in the file. Save it by pressing `ctrl + x` followed by an `enter`. And close the editor by pressing `ctrl + o`. 
6. Enable the systemd service by running the following command: `sudo systemctl enable ngrok.service`
7. Start the systemd service by running the following command: `sudo systemctl start ngrok.service`

For more information, check these two resources:
* Ngrok download page: https://ngrok.com/download
* Ngrok easy guide for linux: https://ngrok.com/docs/guides/device-gateway/linux/
* Ngrok configuration and systemd service instructions: https://stackoverflow.com/questions/50681671/how-to-keep-ngrok-running-even-when-signing-off-of-a-server

#### SSH into a server:

Use the following command to SSH into a server. Notice that the `-p` here is small 'p', unlike how `-P` is used to denote a port in the `scp` command, which is a capital 'P'.
```
ssh -p port_number username@server_address
```

For example: 
```
ssh -p 10609 rootuser@0.tcp.in.ngrok.io 
```

Get this port and the IP from https://dashboard.ngrok.com/tunnels/agents. For me, the username of my linux user is rootuser. So I log into that first. Then I access the root using `sudo su`. I can not log into root directly. 

After entering this command, you will be prompted with: 
```
The authenticity of host '[0.tcp.in.ngrok.io]:10609 ([3.6.122.107]:10609)' can't be established.
ECDSA key fingerprint is SHA256:B5****************w4Y.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

Type `yes` and hit enter. It should next ask you for your password. In most cases this should be the same password as your root password. 

#### Sending a file from PC to the server: 

Use the SCP command to send a file from PC to the server.
```
scp -P port [options] source_file destination_file
```

Here are some additional options that you can use with the SCP command:

 * `-r`: Recursively copy the entire directory and its contents.
 * `-p`: Preserve the file permissions and timestamps. [This is small p, port is capital P.]
 * `-v`: Verbose mode, which will display more information about the file transfer.

For example, to copy the file `myfile.txt` from your Windows PC to the directory `/home/rootuser/` on the Linux machine, preserving the file permissions and timestamps, and using port 3940 for the SSH connection, you would use the following command:
```
scp -P 3940 -p myfile.txt rootuser@linux_machine_ip:/home/user/
```

#### Downloading a file from the server in PC:

To download the file from the Linux machine to your Windows PC, you can use the SCP command again. The syntax is the same as before, but you need to specify the source file and destination file in reverse. For example, to download the file myfile.txt from the directory /home/user/Documents on the Linux machine to the directory C:\Users\username\Downloads on your Windows PC, you would use the following command:
```
scp rootuser@linux_machine_ip:/home/user/Documents/myfile.txt C:\Users\username\Downloads
```

You can use the `-P` option to specify the port of the linux machine here as well. 

#### Local Network NAS: 

To have your own network attached storage, you need to install Samba on your linux system. But remember, **this NAS can not be accessed outside local network without port forwarding**. To install samba and setup a network drive, follow the steps below. 

1. To install Samba, run: `sudo apt update` and then, `sudo apt install samba`.
2. Check if the installation was successful by running: `whereis samba`. The output should look like: `samba: /usr/sbin/samba /usr/lib/samba /etc/samba /usr/share/samba /usr/share/man/man7/samba.7.gz /usr/share/man/man8/samba.8.gz`.
3. Now that Samba is installed, we need to create a directory for it to share: `mkdir /home/<username>/SharedStorage/`. For example, for me, the command was: `mkdir /home/rootuser/SharedStorage/`
4. The configuration file for Samba is located at `/etc/samba/smb.conf`. To add the new directory as a share, we edit the file by running: `sudo nano /etc/samba/smb.conf` At the bottom of the file, add the following lines:
    ```
    [shared]
        comment = Samba on Ubuntu
        path = /home/username/SharedStorage
        read only = no
        browsable = yes
    ```
5. Then press `ctrl + o` to save and `ctrl+x` to exit from the nano text editor.
6. Now that we have our new share configured, restart Samba for it to take effect: `sudo service smbd restart`. 
7. Update the firewall rules to allow Samba traffic: `sudo ufw allow samba`.
8. We need to set up a Samba password for our user account: `sudo smbpasswd -a username`. Note that, username used must belong to a system account, else it won’t save. For me, the command was: `sudo smbpasswd -a rootuser`. Make sure this password contains **no special characters or capital letters** (not sure about the capital letters).
9. To make samba run every time your machine boots up, run: `sudo systemctl enable smbd` (I have not tested this).
9. Run this command to see your IP address: `ifcofig`. Copy the IP address that looks like `192.168.xxx.xxx`.
10. To connect the drive to a windows PC, ope file manager, to go This PC. Go to Map netowrk drive -> Map network drive. Put the IP address you copied on the followig format: `\\ip-address\shared`. Here, `shared` is the string you wrote inside the square brackets in the config file. For me, it was `\\192.168.25.96\share`. Check the box 'Connect with different credentials' and hit 'Finish'. You will be prompted for a password. Type in the samba password you created. 

For more information: https://ubuntu.com/tutorials/install-and-configure-samba#3-setting-up-samba

Common troubleshooting: 
* Windows connection issue: https://stackoverflow.com/questions/58528469/windows-10-cant-connect-to-ubuntu-16-04-samba
* Windows password issue: https://serverfault.com/questions/549289/the-network-folder-specified-is-currently-mapped-using-a-different-user-name-and

#### File transfer by SCP over SSH using WinSCP: 

Download and install WinSCP on Windows. Then follow the following steps: 

1. Open WinSCP. Click on new tab. 
2. Put the following values in the following fields. 
	* File protocol: SCP
	* Hostname: The ngrok address without the `tcp://` part
	* Port: ngrok port
	* Username and password: your user's username and password (not the Samba one).
3. Save the configuration and connect. 
4. On the left you will see your Windows files, on the right you will see the files and directories in your linux system. 

When you transfer to or from a directory in linux, you might see and error such as, you don't have sufficient permissions for this. This occurs when your linux user does not have permission to a specific directory or file. Using the following command can give permission to an user: `chmod -R <permissions> <directory>`. To give all users complete permissions to a directory, use `chmod -R 777 <directory>`. Here, `-R` is for recursive. It gives permission for all files, directories inside the directory mentioned. 

Some more information: https://superuser.com/questions/311658/make-a-network-drive-available-over-the-internet


### Alternative Services

#### Installing NodeJs and NPM (huge pain):

This step is only required if you want to use Localtunnel. Doing apt-get install nodejs always installs version 10.x. But we need the latest version. So here's how you achieve that. 

1. Go to NodeJs official download site (https://nodejs.org/en/download/current). 
2. Find 'Linux Binaries (x64)', right click on the download link (currently it's just a button with '64-bit' written on it) and copy the link. 
3. Go to your ubuntu terminal, run the following: `wget <link you just copied>`. For example: `wget https://nodejs.org/dist/v21.2.0/node-v21.2.0-linux-x64.tar.xz`. Make sure to get the ARM link for ARM systems. You can know what system you are on by running, `uname -m`. If it shows something like `x86_64` then you are not on an ARM system. 
4. Make a directory where you want to install NodeJs and your other dowloaded softwares in. I dowloaded the archive in my /home/user/ (the user directory you end up in after you run `sudo su`) and I created a directory in it to install NodeJs and other downloaded softwares in. You can do it using `mkdir <directory name>`. For example, I ran `mkdir installed-softwares`. 
5. Extract the files using this command: `sudo tar -xJvf <nodejs package filename> -C <installation-directory>`. For example, my command was: `sudo tar -xJvf node-v21.2.0-linux-x64.tar.xz -C installed-softwares/`. Run this command with `-xJf` instead of `-xJvf` to not show logs during the extraction process.
6. Go to installed-softwared: `cd installed-softwares`.
7. Check the files and folders there: `ls`.
8. You should see a directory named `node-v21.2.0-linux-x64`. Go in it: `cd node-v21.2.0-linux-x64`. Check the files ad directories inside it usign the `ls` command. There should a `bin` directory. The whole path to bin should be something like `/home/rootuser/installed-softwares/node-v21.2.0-linux-x64`. We will need this directory location later. 
9. Open your shell configuration file for editing. For example: `nano ~/.bashrc` or `nano ~/.bash_profile`. Choose the appropriate file based on your shell. For me `nano ~/.bashrc` worked, so I recommed that. 
10. Add the following line in the very bottom on the file: `export PATH=<nodejs installation directory>/bin:$PATH`. For me, this was: `export PATH=/home/rootuser/installed-softwares/node-v21.2.0-linux-x64/bin:$PATH`
11. Save the file by pressig `ctrl + o` and exit from nano editor by pressing `ctrl + x`. 
12. To apply the changes to your current session, you can either restart your terminal or run: `source ~/.bashrc`. 
13. Lastly, check the installation by running: `node -v`.

For more info: https://github.com/nodejs/help/wiki/Installation

#### Installing Localtunnel: 

##### Caution: 

Localtunnel does not have a web UI like ngrok. So if you are running an SSH server on a machine without a monitor, you won't be able to know what address your localtunnel service is available at. **You can't use localtunnel for SSH**. But you can use localtunnel on a machine where ngrok is already running, and use localtunnel to expose an apache webserver or something to the web. But, opening a localtunnel address on a browser prompts you to put the IP of the host machine first. This is a headache sometimes. So, just use bore instead (instructions below). 

After you have installed NodeJs and npm (npm comes with NodeJs), run this command to install localtunnel.
```
npm install -g localtunnel
```

To expose a port to the web, run this command:
```
lt --port <port>
```

For example, to expose port 8000, run:
```
lt --port 8000
```

#### Installing Bore (best ngrok alternative):

##### Caution: 

Bore does not have a web UI like ngrok. So if you are running an SSH server on a machine without a monitor, you won't be able to know what address your bore service is available at. For SSH server purposes, ngrok is still the best. But you can use bore on a machine where ngrok is already running, and use bore to expose an apache webserver or something to the web. 

Bore installation is very similar to NodeJs installation.

1. Go to bore official GitHub page (https://github.com/ekzhang/bore). 
2. Go to the Releases page (https://github.com/ekzhang/bore/releases), and find the release that suits you. For me, it is the 64-bit Linux release. Make sure to get the ARM link for ARM systems. You can know what system you are on by running `uname -m`. If it shows something like `x86_64` then you are not on an ARM system. Currently, the filename for 64-bit linux systems looks like `bore-v0.5.0-x86_64-unknown-linux-musl.tar.gz`. Right click on it and copy the link.
3. Go to your ubuntu terminal, run the following: `wget <link you just copied>`. For example: `wget bore-v0.5.0-x86_64-unknown-linux-musl.tar.gz`.  
4. Make a directory where you want to install bore and your other dowloaded softwares in. I dowloaded the archive in my /home/user/ (the user directory you end up in after you run `sudo su`) and I created a directory in it to install bore and other downloaded softwares in. You can do it using `mkdir <directory name>`. For example, I ran `mkdir installed-softwares`. Skip this step if you have already done this earlier. 
5. Make a directory specifically for bore inside that directory you created in the previous step. This is because extracting bore package just extracts a runnable file, that's not in any directory. Run `mkdir <your softwares directory>/bore/`. For me, this command was: `mkdir installed-softwares/bore`.
6. Extract the files using this command: `sudo tar -xzvf <nodejs package filename> -C <installation-directory>`. For example, my command was: `sudo tar -xJvf bore-v0.5.0-x86_64-unknown-linux-musl.tar.gz -C installed-softwares/bore/`. Run this command with `-xzf` instead of `-xzvf` to not show logs during the extraction process.
7. Go to installed-softwared: `cd installed-softwares`.
8. Check the files and folders there: `ls`.
9. You should see a directory named `bore`, that you created earlier. Go in it: `cd bore`. Check the files and directories inside it usign the `ls` command. There should a runnable file `bore`. The whole path to bore directory should be something like `/home/rootuser/installed-softwares/bore`. We will need this directory location later. 
10. Open your shell configuration file for editing. For example: `nano ~/.bashrc` or `nano ~/.bash_profile`. Choose the appropriate file based on your shell. For me `nano ~/.bashrc` worked, so I recommed that. 
11. Add the following line in the very bottom on the file: `export PATH=<bore installation directory>:$PATH`. For me, this was: `export PATH=/home/rootuser/installed-softwares/bore:$PATH`
12. Save the file by pressig `ctrl + o` and exit from nano editor by pressing `ctrl + x`. 
13. To apply the changes to your current session, you can either restart your terminal or run: `source ~/.bashrc`. 

To use bore, run the following command:
```
bore local <port> --to bore.pub
```

This would expose the localhost on the port specified to the web. Bore will print a randomly generated port number (example: `bore.pub:18989`). Use this address to access your `localhost:<port>` on the web. 

To expose a webserver running on port 8000, run:
```
bore local 8000 --to bore.pub
```

#### Conclusion

If this content helped you, please give a star to this repository. Feel free to clone this repository. PRs are also welcome. 

A very small portion of the content here is copied from other sources. I hope it doesn't fall under piracy as I only intend to share the kowledge through this writeup. 