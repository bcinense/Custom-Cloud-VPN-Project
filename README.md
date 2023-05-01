# Custom Cloud VPN Project - ITM684
# Pre-requisites
### Step 1. Create a Digital Ocean Account
- I used my @hawaii.edu account

### Step 2. Create a new droplet and name it **OPENVPNserver**. Make the selections below when creating the droplet:
- Reference to initial server set-up: [Initial Server Setup with Ubuntu 22.04](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-22-04) 
	- Region - New York
	- Datecenter - NYC1
	- Image - Ubuntu
	- Version - 22.04 (LTS) x64
	- Size - Basic at $6/month
	- SSH or Password (I used password)

### Step 3. Open the console for the OPENVPNserver droplet and log in as root
`ssh root@server_ip_address`

### Step 4. Create new user and grant administrative priveleges
`adduser username_here`<br>
`usermod -aG sudo username_here`

### Step 5. Set up Firewall and change default SSH port to 41235
`ufw app list`<br>
`ufw allow OpenSSH`<br>
`ufw enable`<br>
- Switch to user account
  `su username_here`<br>
  `sudo ufw allow 41235`<br>
  `sudo nano /etc/ssh/sshd_config`<br>
- Search for #port 22, remove the # and change the port number to 41235<br>
	`sudo systemctl restart ssh`<br>
- Verify ssh is listening<br>
	`ss -an | grep 41235`<br>
- Log out of the server and ssh back in using the command<br>
	`ssh username_here@server_ip_address -p41235`

### Step 6. Create a second new droplet and name it **CAserver** and follow Step 2 to Step 5 above
- Reference to initial server set-up: [Initial Server Setup with Ubuntu 22.04](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-22-04) 

### Step 7. (In the CAserver droplet) Set-up and Configure a Certificate Authority
- Reference: [How To Set Up and Configure a Certificate Authority (CA) on Ubuntu 22.04](https://www.digitalocean.com/community/tutorials/how-to-set-up-and-configure-a-certificate-authority-on-ubuntu-22-04)

  - Step 7a. Install Easy-Rsa<br>
	`sudo apt update`<br>
	`sudo apt install easy-rsa`
  - Step 7b. Prepare a Public Key Infrastructure Directory<br>
	`mkdir ~/easy-rsa`<br>
	`ln -s /usr/share/easy-rsa/* ~/easy-rsa/`<br>
	`chmod 700 /home/username_here/easy-rsa`<br>
	`cd ~/easy-rsa`<br>
	`./easyrsa init-pki`

  - Step 7c. Create a Certificate Authority<br>
	`cd ~/easy-rsa`<br>
	`nano vars`<br>
	  - Copy and paste the text below in the file, CTRL + X and the ENTER to exit out of the file<br>
      `set_var EASYRSA_REQ_COUNTRY    "US"`<br>
      `set_var EASYRSA_REQ_PROVINCE   "NewYork"`<br>
      `set_var EASYRSA_REQ_CITY       "New York City"`<br>
      `set_var EASYRSA_REQ_ORG        "DigitalOcean"`<br>
      `set_var EASYRSA_REQ_EMAIL      "admin@example.com"`<br>
      `set_var EASYRSA_REQ_OU         "Community"`<br>
      `set_var EASYRSA_ALGO           "ec"`<br>
      `set_var EASYRSA_DIGEST         "sha512"`<br>

  - Step 7d. Distribute the Certificate Authority's Public Certificate<br>
  `./easyrsa build-ca nopass`<br>
	`cat ~/easy-rsa/pki/ca.crt`
	  - Output will look something like this:<br>
		`-----BEGIN CERTIFICATE-----
		MIIDSzCCAjOgAwIBAgIUcR9Crsv3FBEujrPZnZnU4nSb5TMwDQYJKoZIhvcNAQEL
		BQAwFjEUMBIGA1UEAwwLRWFzeS1SU0EgQ0EwHhcNMjAwMzE4MDMxNjI2WhcNMzAw
		. . .
		. . .
		-----END CERTIFICATE-----`
	      - Copy everything, including the dashes and text

	Step 7e. (Switch to OPENVPNserver) Open a file<br>
	`nano /tmp/ca.crt`
	- Paste the text from the CAserver (from Step 7d) into this file, CTRL + X and then ENTER to exit file. Then:<br>
	`sudo cp /tmp/ca.crt /usr/local/share/ca-certificates/`<br>
	`sudo update-ca-certificates`

# Set-up and Configure OPENVPN Server
#### Reference: https://www.digitalocean.com/community/tutorials/how-to-set-up-and-configure-an-openvpn-server-on-ubuntu-22-04
### Step 1. (In droplet OPENVPNserver)  Install OPENVPN and Easy-RSA
`sudo apt update`<br>
`sudo apt install openvpn easy-rsa`<br>
`mkdir ~/easy-rsa`<br>
`ln -s /usr/share/easy-rsa/* ~/easy-rsa/`<br>
`sudo chown sammy ~/easy-rsa`<br>
`chmod 700 ~/easy-rsa`<br>

### Step 2. Create a PKI for OPENVPN
`cd ~/easy-rsa`<br>
`nano vars`<br>
- Copy and paste the text below into the file:<br>
	`set_var EASYRSA_ALGO "ec"`<br>
	`set_var EASYRSA_DIGEST "sha512"`<br>
`./easyrsa init-pki`

### Step 3. Create and OPENVPN Server Certificate Request and Private Key
`cd ~/easy-rsa`<br>
`./easyrsa gen-req server nopass`<br>
- Press enter to default the Common Name<br>
`sudo cp /home/sammy/easy-rsa/pki/private/server.key /etc/openvpn/server/`

### Step 4. Sign the OPENVPN Server Certificate Request
- First go to the droplet name OPENVPNserver:<br>
`scp -P 41235 /home/username_here/easy-rsa/pki/reqs/server.req username_here@your_ca_server_ip:/tmp`<br>
- Then go to the droplet name CAserver:<br>
`cd ~/easy-rsa`<br>
`./easyrsa import-req /tmp/server.req server`<br>
`./easyrsa sign-req server server`<br>
`scp -P 41235 pki/issued/server.crt username_here@your_vpn_server_ip:/tmp`<br>
`scp -P 41235 pki/ca.crt username_here@your_vpn_server_ip:/tmp`<br>
- Switch back to the droplet name OPENVPNserver:<br>
`sudo cp /tmp/{server.crt,ca.crt} /etc/openvpn/server`<br>

### Step 5. Configure OPENVPN Cryptographic Material
- In droplet name OPENVPNserver<br>
`cd ~/easy-rsa`<br>
`openvpn --genkey --secret ta.key`<br>
`sudo cp ta.key /etc/openvpn/server`<br>

### Step 6. Generate a Client Certificate and Key Pair
- In droplet name OPENVPNserver<br>
`mkdir -p ~/client-configs/keys`<br>
`chmod -R 700 ~/client-configs`<br>
`cd ~/easy-rsa`<br>
`./easyrsa gen-req client1 nopass`<br>
`cp pki/private/client1.key ~/client-configs/keys/`<br>
`scp -P 41235 pki/reqs/client1.req username_here@your_ca_server_ip:/tmp`<br>
- In droplet name CAserver<br>
`cd ~/easy-rsa`<br>
`./easyrsa import-req /tmp/client1.req client1`<br>
`scp -P 41235 pki/issued/client1.crt username_here@your_server_ip:/tmp`<br>
- In droplet name OPENVPNserver<br>
`cp /tmp/client1.crt ~/client-configs/keys/`<br>
`cp ~/easy-rsa/ta.key ~/client-configs/keys/`<br>
`sudo cp /etc/openvpn/server/ca.crt ~/client-configs/keys/`<br>
`sudo chown username_here.username_here ~/client-configs/keys/*`<br>

## IMPORTANT NOTE
Before Step 7, you can now power off the CAserver droplet and use the OPENVPNserver droplet for the rest of the steps<br>
- In CAserver droplet enter;<br>
`shutdown`

### Step 7. Configuring OPENVPN
- In droplet name OPENVPNserver<br>
`sudo cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf /etc/openvpn/server/`<br>
`sudo nano /etc/openvpn/server/server.conf`<br>
- Search for the tls-auth directive. This line will be enabled by default. Comment it out by adding a ; to the beginning of the line. Then add a new line after it containing the value tls-crypt ta.key only:<br>
	![image](https://user-images.githubusercontent.com/46617761/235441722-8d5d954d-7937-4b66-834a-25f254a85258.png)<br>
- The default value is set to AES-256-CBC, however, the AES-256-GCM cipher offers a better level of encryption, performance, and is well supported in up-to-date OpenVPN clients. Comment out the default value by adding a ; sign to the beginning of this line, and then add another line after it containing the updated value of AES-256-GCM:<br>
	![image](https://user-images.githubusercontent.com/46617761/235441758-02a08353-7c57-4043-8225-1622daabf339.png)<br>
- Right after the cipher AES-256-GCM add auth SHA256:<br>
	![image](https://user-images.githubusercontent.com/46617761/235441870-ff4b0ba3-73b9-4f4c-96e6-10743ee2f433.png)<br>
- Add auth SHA256 and Comment out the existing line that looks like dh dh2048.pem or dh dh.pem. The filename for the Diffie-Hellman key may be different than what is listed in the example server configuration file. Then add a line after it with the contents dh none:<br>
	![image](https://user-images.githubusercontent.com/46617761/235441831-38c110f1-acfc-411e-ad2b-35f558b27620.png)
- Uncomment user nobody and group nogroup (this may say nobody, change it to nogroup)<br>
	![image](https://user-images.githubusercontent.com/46617761/235441904-3dabfd90-c50f-48fd-b2ad-caf0b0eae7e2.png)<br>
- Uncomment:<br>
	![image](https://user-images.githubusercontent.com/46617761/235441938-4eb743bf-84bd-43b3-b240-4b3933858d99.png)<br>
	![image](https://user-images.githubusercontent.com/46617761/235441956-b6e42ba5-abcf-43a0-9ae3-a4aa56eca222.png)<br>
- Adjust port from port 1194 to port 443 amd change to proto tcp<br>
- Since we are switching from UDP to TCP, change the explicit-exit-notify directive’s value from 1 to 0<br>
	![image](https://user-images.githubusercontent.com/46617761/235441986-fae30ce7-4174-4cb3-a06c-38f0d75e7775.png)<br>

### Step 8. Adjust the OPENVPN Server Networking Configuration
`sudo nano /etc/sysctl.conf`<br>
- Add the following to the end of the file:<br>
	net.ipv4.ip_forward = 1<br>
- To check:<br>
	`sudo sysctl -p`<br>

### Step 9. Firewall Configuration
`ip route list default`<br>
`sudo nano /etc/ufw/before.rules`<br>
- Towards the top of the file, add the highlighted lines below. This will set the default policy for the POSTROUTING chain in the nat table and masquerade any traffic coming from the VPN. Remember to replace eth0 in the -A POSTROUTING line below with the interface you found in the above command:<br>
			`# START OPENVPN RULES<br>
			# NAT table rules<br>
			*nat<br>
			:POSTROUTING ACCEPT [0:0]<br>
			# Allow traffic from OpenVPN client to eth0 (change to the interface you discovered!)<br>
			-A POSTROUTING -s 10.8.0.0/8 -o eth0 -j MASQUERADE<br>
			COMMIT<br>
			# END OPENVPN RULES`<br>
`sudo nano /etc/default/ufw`<br>
- Inside, find the DEFAULT_FORWARD_POLICY directive and change the value from DROP to ACCEPT:<br>
	![image](https://user-images.githubusercontent.com/46617761/235442170-fca2e615-544c-4877-8d04-36eced6d2640.png)<br>
`sudo ufw allow 443/tcp`<br>
`sudo ufw allow OpenSSH`<br>
`sudo ufw allow 53/udp`<br>
`sudo ufw allow OpenSSH`<br>

### Step 10. Starting OPENVPN
`sudo systemctl -f enable openvpn-server@server.service`<br>
`sudo systemctl start openvpn-server@server.service`<br>
- Check to see if it running<br>
`sudo systemctl status openvpn-server@server.service`

### Step 11. Create Client Configuration Infrastructure
`mkdir -p ~/client-configs/files`<br>
`cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf ~/client-configs/base.conf`<br>
`nano ~/client-configs/base.conf`<br>
- In the file replace the text after remote with server IP and port 443 and proto tcp:<br>
	![image](https://user-images.githubusercontent.com/46617761/235442241-5efdce34-54cf-4662-8a78-155b342e54cb.png)<br>
- Uncomment:<br>
	![image](https://user-images.githubusercontent.com/46617761/235442294-da6d69f0-a502-4c1b-8ae8-a28200062721.png)<br>
- Comment out:<br>
	![image](https://user-images.githubusercontent.com/46617761/235442327-d600e6cf-9bfd-4a42-8d39-7b9c4d2ec0a6.png)<br>
	![image](https://user-images.githubusercontent.com/46617761/235442349-2084392d-cf97-475c-911f-bb5ab9633efc.png)<br>
- Add:<br>
	![image](https://user-images.githubusercontent.com/46617761/235442373-63f55e10-c791-44ef-bc32-05675106918f.png)<br>
	![image](https://user-images.githubusercontent.com/46617761/235442392-e578fdbf-ae1e-4059-b06e-a38305adba9c.png)<br>
- Add these uncommented out lines anywhere:<br>
	`; script-security 2<br>
	; up /etc/openvpn/update-resolv-conf<br>
	; down /etc/openvpn/update-resolv-conf`<br>
- Create a script:<br>
`nano ~/client-configs/make_config.sh`<br>
	- Add the content below:<br>
	![image](https://user-images.githubusercontent.com/46617761/235442731-21f4129a-fb06-4b34-afdd-eb7d4fecb859.png)<br>
- Make executable:<br>
	`chmod 700 ~/client-configs/make_config.sh`

### Step 12. Generate Client Configuration 
`cd ~/client-configs`<br>
`./make_config.sh client1`<br>
`ls ~/client-configs/files`<br>
- You need to transfer this file to the device you plan to use as the client. For instance, this could be your local computer or a mobile device.<br> **Execute command on local terminal (I have a mac)<br>
`sftp -P 41235 username_here@openvpn_server_ip:client-configs/files/client1.ovpn ~/`<br>

### Step 13. Install Client Configurations
macOS - [Tunnelblick Downloads page](https://tunnelblick.net/downloads.html) <br>
- Double-click the .dmg file and folow the prompts to install<br>
	- Towards the end of the installation process, Tunnelblick will ask if you have any configuration files. Answer I have configuration files and let Tunnelblick finish. Open a Finder window and double-click client1.ovpn. Tunnelblick will install the client profile. Administrative privileges are required.<br>
		- Connecting: Launch Tunnelblick by double-clicking the Tunnelblick icon in the Applications folder. Once Tunnelblick has been launched, there will be a Tunnelblick icon in the menu bar at the top right of the screen for controlling connections. Click on the icon, and then the Connect client1 menu item to initiate the VPN connection. If you are using custom DNS settings with Tunnelblick, you may need check “Allow changes to manually-set network settings” in the advanced configuration dialog.

### Step 14. Test VPN Connection
To test connection go to  [DNSLeakTest](https://www.dnsleaktest.com/)<br>
- The site will return the IP address assigned by your internet service provider and as you appear to the rest of the world. To check your DNS settings through the same website, click on Extended Test and it will tell you which DNS servers you are using.<br>
	- Now connect the OpenVPN client to your Droplet’s VPN and refresh the browser. A completely different IP address (that of your VPN server) should now appear, and this is how you appear to the world. Again, DNSLeakTest’s Extended Test will check your DNS settings and confirm you are now using the DNS resolvers pushed by your VPN.


# Deploy the Algo Server
Link reference for deploying the slgo server : https://github.com/trailofbits/algo <br>

From mac terminal;<br>
`ssh user@hostname -p 41235`<br>
`sudo usermod -aG sudo username`<br>

### Step 1. Get a copy of the Algo scripts<br>
`git clone https://github.com/trailofbits/algo.git`<br>

### Step 2. Install Algo dependicies<br>
`sudo apt install -y --no-install-recommends python3-virtualenv`<br>

### Step 3. `cd algo`<br>

### Step 4. Install the remaing Algo dependencies<br>
`sudo python3 -m virtualenv --python="$(command -v python3)" .en`<br>
`source .env/bin/activate`<br>
`python3 -m pip install -U pip virtualenv`<br>
`python3 -m pip install -r requirements.txt`

### Step 5. `./algo`
#### Step 5a. Update python to python 3.8
`sudo add-apt-repository ppa:deadsnakes/ppa`<br>
`sudo apt-get update`<br>
`sudo apt-get install python3.8`
#### Step 5b. Update path
`echo 'export PATH="/usr/bin:/usr/local/bin:$PATH"' >> ~/.bashrc`<br>
`source ~/.bashrc`
#### Step 5c. Run ./algo again
`./algo`
### Step 6. Firewall needs to be restarted
`sudo ufw enable`
### 7. Import conf file to Wireguard on your local machine
`scp -P 41235 username@<IP>:~/algo/configs/<IP>/wireguard/username.conf ~/Downloads/`
### Step 8. Open Wireguard and select the file copied from the server to the local terminal in step 8

# Add **color coding** to the terminal
### Step 1. Modifying the PS1 environment variable, which controls the prompt that appears in the terminal. Open the .bashrc file in your home directory using a text editor such as nano or vim<br>
`nano ~/.bashrc`
### Step 2. Scroll to the bottom of the file and add the following lines:<br>
`# Define some colors`<br>
`RED="\[\033[0;31m\]"`<br>
`GREEN="\[\033[0;32m\]"`<br>
`YELLOW="\[\033[0;33m\]"`<br>
`BLUE="\[\033[0;34m\]"`<br>
`PURPLE="\[\033[0;35m\]"`<br>
`CYAN="\[\033[0;36m\]"`<br>
`WHITE="\[\033[0;37m\]"`<br>
`RESET="\[\033[0m\]"`<br>
	`#Set the prompt to include colors`<br>
`PS1="${GREEN}\u${YELLOW}@${RED}\h${YELLOW}:${BLUE}\w${RESET}$ "`<br>

# Add aliases
### Go into file to add aliases
- `nano ~/.bashrc`<br>
- Add following aliases: <br>
	Clear command, which clears the screen<br>
		`alias c='clear'`<br>
	Update in one command<br>
		`alias update='sudo apt-get update && sudo apt-get upgrade'`<br>
	Get server cpu info<br>
		`alias cpuinfo='lscpu'`<br>
- Then reload the file with aliases saved and ready to execute:<br>
`source ~/.bashrc`<br>
- To show all aliases<br>
	`aliases`<br>
#### Important note - When adding aliases do not put spaces before or after the equal sign

# Creating 3 cronjobs
### Step 1. Edit the cron jobs list by running the following command:<br>
		`sudo crontab -e`<br>
### Step 2. Select nano<br>
![image](https://user-images.githubusercontent.com/46617761/235439750-3743aa90-da7f-49b7-a451-4fa8f48739ec.png)<br>

### Step 3. Add cronjob lines<br>
- CRONJOB 1<br>
			- In the editor, add the following line to schedule the system updates for every Sunday at midnight:<br>
			`0 0 * * 0 apt update && apt upgrade -y`<br>
- CRONJOB 2<br>
			- This cron job will send an email reminder every Friday at 5pm.<br>
			`0 17 * * 5 echo "Don't forget to submit your Custom Cloud VPN Project!" | mail -s "Reminder" user@example.com`<br>
- CRONJOB 3<br>
			- This cron job will clean up old log files every day at 3am.<br>
			`0 3 * * * find /var/log -type f -mtime +7 -exec rm {} \;`<br>



# In the video - 
-   Launch a browser and navigate to your Github-pages hosted installation documentation 
-   VM operation and configuration: Show the following in your video
-   Log into the server
-   Launch a shell and show the IP address of your VM
-   Show the user list
	- `cut -d: -f1 /etc/passwd | grep -vE '^#|^$'`
-   Show the sudoers
	- `getent group sudo`
-   Demonstrate some of your aliases
	- `c`
	- `update`
	- `cpuinfo`
-   ssh into Sal's DigitalOcean server (he will provide login credentials separately)
-   show any cool customizations (if any) you did 
	- Adding the color coding
-   show the three cronjobs you included and discuss what they do; show related scripts
-   show yourself connecting to your custom cloud VPN server using OpenVPN and Wireguard clients and also IPSec from your main laptop
