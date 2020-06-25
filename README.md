# upload2nc
Send scanned documents from Brother to Nextcloud
## Scan from Brother Multifunction Scanner/Printer to Nextcloud
The Brother Multi device I have (DCP-L2560DW) does not support Nextcloud as
target, but luckily it supports FTP upload. I also have an OpenWRT based router which is always powered on,
so the idea is to turn the OpenWRT router to a gateway between the scanner and Nextcloud by setting up following services on it:
- FTP server, to receive files from the Scanner
- A custom upload script which is triggered by files received via FTP. This will post the file to a Nextcloud share. 
## Advantages
- No adminisrative rights on the Nextcloud server needed.
- No need to run scheduled folder scan on Nexcloud server, the uploaded file will be shown immediately
- No need to install ftp/samba server on the Nextcloud server
- The procedure could be easily adjusted to upload via Samba instead of FTP, in case your scanner does not support upload via FTP

## Security WARNING
- FTP is an unsecure protocol (username/password are sent in clear text over the network), but I think that it is ok for my home network. To mitigate the risks one can set up a separate VLAN just for the scanner, but it is out of the scope of this howto.
- The file uploaded by the scanner to the OpenWRT will be sent to the Netxtcloud server via a secure channel if your Nextcloud URL starts with https://

## Requirements
- root ssh access to your OpenWRT router to install required packages and script
- user access to your Nextcloud to create a share for scanned documents
- administrative access to your Brother Scanner to create FTP upload profile

## Settings on Nextcloud
Set up a file drop upload share as described here: https://docs.nextcloud.com/server/18/user_manual/files/file_drop.html
Password is optional as the share permissions will not let anyone list or download files.
Save the share link, you'll need it later. 

Example share link: https://mycloud.mydomain.net/s/N3XtCl0dShAr31D

Nextcloud URL: https://mycloud.mydomain.net

Share id: N3XtCl0dShAr31D

## Settings on OpenWRT router
### Login and update package manager
```bash
opkg update
```
#### Install FTP server
```bash
opkg install vsftpd
```
#### Configure FTP server
Check if uid 1000 is not taken by existing users
```bash
grep 1000 /etc/passwd
```
Create a system user for FTP use and set it's password
```bash
echo 'brother:x:1000:55::/tmp/tmp:/bin/false' >>/etc/passwd
passwd brother
```
Enable logging and start FTP server
```bash
sed -i 's/#syslog_enable=YES/syslog_enable=YES/' /etc/vsftpd.conf
/etc/init.d/vsftpd enable
/etc/init.d/vsftpd start
```
#### Add firewall rule to let FTP traffic in
- Replace 'IP.OF.BRO.THER' with the ip of your scanner
- Replace 'lan' with the network segment of the scanner. If you do not have an exotic configuration with many network segments added, then leave it as 'lan'.
```bash
uci add firewall rule
uci set firewall.@rule[-1].name='Allow FTP from Brother2'
uci set firewall.@rule[-1].src='lan'
uci set firewall.@rule[-1].dest_port='20-21'
uci set firewall.@rule[-1].proto='tcp'
uci set firewall.@rule[-1].family='ipv4'
uci set firewall.@rule[-1].src_ip='IP.OF.BRO.THER'
uci set firewall.@rule[-1].target='ACCEPT'
uci commit firewall
/etc/init.d/firewall reload
```
### Set up the upload2nc script
#### Install required packages
```bash
opkg install inotifywait curl
```
#### Create config for upload2nc
- Replace N3XtCl0dShAr31D with the 15 characters long Share id you saved after creating the share
- Replace 'ThIsIsOpTiOnAl' with the share password, otherwise it should be empty: '' (two single apostrophes)
- Replace https://nextcloud.server.url with your Nextclour URL. No trailing slash is needed.
```bash
cd /etc/config
touch upload2nc
uci add upload2nc brother
uci set upload2nc.brother.tmpdir=/tmp/tmp
uci set upload2nc.brother.ncurl=https://nextcloud.server.url
uci set upload2nc.brother.ncshare=N3XtCl0dShAr31D
uci set upload2nc.brother.ncsharepw='ThIsIsOpTiOnAl'
uci commit upload2nc
```
#### Copy upload2nc to Openwrt
Download the script to /etc/init.d/upload2nc and set permissions
```bash
wget https://raw.githubusercontent.com/ZsZs73/upload2nc/master/upload2nc -O /etc/init.d/upload2nc
chmod 755 /etc/init.d/upload2nc
```
#### Enable and start the script
```bash
/etc/init.d/upload2nc enable
/etc/init.d/upload2nc start
```
## Settings on the Scanner
### Create an Ftp profile
- Log in to the device at https://IP.OF.BRO.THER
- Navigate to Scan tab / Scan to FTP
- Add a user defined filename in case you do not like any of the default ones
  - Enter the new filename prefix to the list position 8 and click Submit
	An 8 digit document counter number will be appended to the prefix
- Navigate to Scan tab / Scan to FTP Profile
- Click on an unused profile and set following
  - Profile Name: Nextcloud (This name will show up on the display of the scanner)
  - Host Address: ip address on your OpenWRT router
  - Username: the FTP username you created on the router (brother)
  - Password: Password of the FTP user (the same as you typed in after 'passwd brother')
  - Store Directory: /tmp/tmp
  - File Name: Select the filename from the list (the prefix you created previously should appear on the list)
  - Quality: set it as you prefer
  - File Type: set it as you prefer
  - Glass Scan Size: set it as you prefer
  - File Size: set it as you prefer
  - Remove Background Color: set it as you prefer or leave it default
  - Passive mode: set it to Off
  - Port number: leave it default: 21
	- Click Submit button
	- Click yes on "Are you sure that you want to test?" -> Green Test OK should show up
### Create a shortcut
- Select 'Scan' on the touch screen of the scanner
- Select 'to Ftp'
- Select 'Nextcloud'
- Save as Shortcut
### Start Scanning
- Select the freshly created shortcut and start scanning

## Troubleshooting
### upload2nc logs
The script creates some log entries about successful/failed uploads
```bash
logread | grep upload2nc
```
### FTP logs
Check FTP logs by
```bash
logread | grep vsftp
```
### Trace network issues (advanced)
Install and run tcpdump. It might help revealing firewall issues, etc
```bash
opkg install tcpdump
tcpdump -nA host IP.OF.BRO.THER and port 21
```
## OpenWrt upgrade
On sysupgrade the content of the flash gets overwritten, so re-installing of the script is needed by executing steps in the following sections above:
- Copy upload2nc to Openwrt
- Enable and start the script

Comments, improvements are welcome.
