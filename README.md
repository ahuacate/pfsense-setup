# pfSense - Setup
This recipe is for setting up pfSense. The following assumes your are familiar with pfSEnse and its preference menus and configuration options.

Network prerequisites are:
- [x] Layer 2 Network Switches
- [x] Network Gateway is `192.168.1.5`
- [x] Network DNS server is `192.168.1.5` (Note: your Gateway hardware should enable you to a configure DNS server(s), like a UniFi USG Gateway, so set the following: primary DNS `192.168.1.254` which will be your PiHole server IP address; and, secondary DNS `1.1.1.1` which is a backup Cloudfare DNS server in the event your PiHole server 192.168.1.254 fails or os down)
- [x] Network DHCP server is `192.168.1.5`

Other Prerequisites are:
- [x] Proxmox node fully configured as per [PROXMOX-NODE BUILDING](https://github.com/ahuacate/proxmox-node/blob/master/README.md#proxmox-node-building)


Tasks to be performed are:
- [1.00 Setup pfSense](#100-setup-pfsense)
	- [1.01 Change Your pfSense Password](#101-change-your-pfsense-password)
	- [1.02 Enable AES-NI](#102-enable-aes-ni)
	- [1.03 Preventing IP address leaks](#103-preventing-ip-address-leaks)
- [2.0 Fix a Static IPv4 and edit Interfaces OPT1, OPT2 WAN](#20-fix-a-static-ipv4-and-edit-interfaces-opt1-opt2-wan)
	- [2.01 Edit Interface OPT1](#201-edit-interface-opt1)
	- [2.02 Edit Interface OPT2](#202-edit-interface-opt2)
	- [2.03 Edit Interface WAN](#203-edit-interface-wan)
- [3.00 Add DHCP Servers to OPT1 and OPT2](#300-add-dhcp-servers-to-opt1-and-opt2)
	- [3.01 Setup DHCP Servers for OPT1 and OPT2](#301-setup-dhcp-servers-for-opt1-and-opt2)
- [4.00 Setup OpenVPN Client Server](#400-setup-openvpn-client-server)
	- [4.01 Add your OpenVPN Client Certificate details](#401-add-your-openvpn-client-certificate-details)
	- [4.02 Create your OpenVPN Client Server(s)](#402-create-your-openvpn-client-servers)
- [5.00 Setting up a gateway for each OpenVPN Client](#500-setting-up-a-gateway-for-each-openvpn-client)
	- [5.01 Add Network Interface Ports](#501-add-network-interface-ports)
	- [5.02 Edit the new Interface Ports](#502-edit-the-new-interface-ports)
	- [5.03 Check your Gateway Interface(s)](#503-check-your-gateway-interfaces)
- [6.00 Create Gateway Groups](#600-create-gateway-groups)
	- [6.01 Create VPNGATEWORLD Gateway Group](#601-create-vpngateworld-gateway-group)
	- [6.02 Create VPNGATELOCAL Gateway Group](#602-create-vpngatelocal-gateway-group)
- [7.00 Adding Firewall Aliases](#700-adding-firewall-aliases)
	- [7.01 Create Firewall Alias - Chromecast IP List](#701-create-firewall-alias---chromecast-ip-list)
- [8.00 Adding NAT Rules](#800-adding-nat-rules)
	- [8.01 Create NAT Rule VLAN30 to vpngate-world01/02 - Outbound](#801-create-nat-rule-vlan30-to-vpngate-world0102---outbound)
	- [8.02 Create NAT Rule VLAN40 to vpngate-local - Outbound](#802-create-nat-rule-vlan40-to-vpngate-local---outbound)
- [9.00 Adding Firewall Rules](#900-adding-firewall-rules)
	- [9.01 Allow Rule for OPT1 - VPNGATEWORLD_GROUP](#901-allow-rule-for-opt1---vpngateworld_group)
	- [9.02 Allow Rule for OPT2 - VPNGATELOCAL_GROUP](#902-allow-rule-for-opt2---vpngatelocal_group)
	- [9.03 Allow OPT2 access to Chromecast and TV devices - vpngate-local](#903-allow-opt2-access-to-chromecast-and-tv-devices---vpngate-local)
	- [9.04 Allow WAN to VLAN30 devices (NZBGet & Deluge) - vpngate-world](#904-allow-wan-to-vlan30-devices-nzbget--deluge---vpngate-world)
	- [9.05 DNS Allow and Block Rules on OPT1 - vpngate-world](#905-dns-allow-and-block-rules-on-opt1---vpngate-world)
	- [9.06 DNS Allow and Block Rules on OPT2 - vpngate-local](#906-dns-allow-and-block-rules-on-opt2---vpngate-local)
- [10.00 Setup pfSense DNS](#1000-setup-pfsense-dns)
	- [10.01 Set Up DNS Resolver](#1001-set-up-dns-resolver)
	- [10.02 Set Up General DNS](#1002-set-up-general-dns)
- [11.00 Install Avahi for mdns](#1100-install-avahi-for-mdns)
	- [11.01 Install Avahi Package](#1101-install-avahi-package)
	- [11.02 Setup Avahi](#1102-setup-avahi)
- [12.00 Setup NTP](#1200-setup-ntp)
- [13.00 Finish Up](#1300-finish-up)
- [14.00 Create a pfSense Backup](#1400-create-a-pfsense-backup)
- [00.00 Patches and Fixes](#0000-patches-and-fixes)
	- [00.01 pfSense – disable firewall with pfctl -d](#0001-pfsense--disable-firewall-with-pfctl--d)



## 1.00 Setup pfSense
You can now access pfSense webConfigurator by opening the following URL in your web browser: http://192.168.1.253/ . In the pfSense webConfigurator we are going to setup two OpenVPN Gateways, namely vpngate-world and vpngate-local. Your default login details are User > admin | Pwd > pfsense

### 1.01 Change Your pfSense Password
Now using the pfSense web interface `System` > `User Manager` > `click on the admin pencil icon` and change your password to something more secure.

Remember to hit the `Save` button at the bottom of the page.

### 1.02 Enable AES-NI 
If your CPU supports AES-NI CPU Crypto (i.e Qotom Mini PC Q500G6-S05 does) best enable it.

Now using the pfSense web interface `System` > `Advanced` > `Miscellaneous Tab` scroll down to the section `Cryptographic & Thermal Hardware` and change the details as shown below:

| Cryptographic & Thermal Hardware | Value | Notes
| :---  | :---: | :--- |
| Cryptographic Hardware | `AES-NI CPU-based Accelleration` | *This works for the Qotom Mini PC Q500G6-S05 series and modern hardware*
| Thermal Sensors | None/ACPI | *Will not work because Proxmox virtualization host will NOT forward CPU temperature data to it's pfSense guest*

Remember to hit the `Save` button at the bottom of the page.

### 1.03 Preventing IP address leaks
This is an important step required to reduce the chance of leaks in the event the VPN goes down for any reason.

Navigate using the pfSense web interface to `System` > `Advanced` and `Miscellaneous Tab`. Scroll down to Gateway Monitoring and set the following:

| Miscellaneous Tab| Value | Notes
| :---  | :---: | :--- |
| **Gateway Monitoring**
| State Killing on Gateway Failure | `[]` Flush all states when a gateway goes down
| Skip rules when gateway is down | `[x]` Do not create rules when gateway is down

And click `Save`

## 2.0 Fix a Static IPv4 and edit Interfaces OPT1, OPT2 WAN
Edit the interfaces as follows:

### 2.01 Edit Interface OPT1
Now using the pfSense web interface `Interfaces` > `OPT1` to open a configuration form, then fill up the necessary fields as follows:

| Interfaces/OPT1 (vtnet2) | Value | Notes
| :---  | :---: | :--- |
| Enable | `☑` | *Check the box*
| Description | `OPT1`
| IPv4 Configuration Type | `Static IPv4`
| Ipv6 Configuration Type | `None`
| MAC Address | Leave blank
| MTU | Leave blank
| MSS | Leave blank
| Speed and  Duplex | `Default (no preference, typically autoselect)`
| **Static IPv4 Configuration**
| IPv4 Address | `192.168.30.5/24`
| IPv4 Upstream gateway | `None`
| **Reserved Networks**
| Block private networks and loopback addresses | `[ ]` | *Uncheck the box*
| Block bogon networks | `☐` | *Uncheck the box*

And click `Save`.

![alt text](https://raw.githubusercontent.com/ahuacate/pfsense-setup/master/images/pfsense_interfaces_01.png)

### 2.02 Edit Interface OPT2
Now using the pfSense web interface `Interfaces` > `OPT2` to open a configuration form, then fill up the necessary fields as follows:

| Interfaces/OPT2 (vtnet3) | Value | Notes
| :---  | :---: | :--- |
| Enable | `☑` | *Check the box*
| Description | `OPT2`
| IPv4 Configuration Type | `Static IPv4`
| Ipv6 Configuration Type | `None`
| MAC Address | Leave blank
| MTU | Leave blank
| MSS | Leave blank
| Speed and  Duplex | `Default (no preference, typically autoselect)`
| **Static IPv4 Configuration**
| IPv4 Address | `192.168.40.5/24`
| IPv4 Upstream gateway | `None`
| **Reserved Networks**
| Block private networks and loopback addresses | `[]` | *Uncheck the box*
| Block bogon networks | `☐` | *Uncheck the box*

And click `Save`.

![alt text](https://raw.githubusercontent.com/ahuacate/pfsense-setup/master/images/pfsense_interfaces_02.png)

### 2.03 Edit Interface WAN
Now using the pfSense web interface `Interfaces` > `WAN` to open a configuration form, then edit the necessary fields to match the following:

| Interfaces/WAN (vtnet1) | Value | Notes
| :---  | :---: | :--- |
| Enable | `☑` | *Check the box*
| Description | `WAN`
| IPv4 Configuration Type | `Static IPv4`
| Ipv6 Configuration Type | `None`
| MAC Address | Leave blank
| MTU | Leave blank
| MSS | Leave blank
| Speed and  Duplex | `Default (no preference, typically autoselect)`
| **Static IPv4 Configuration**
| IPv4 Address | `192.168.2.1/28`
| IPv4 Upstream gateway | Click `Add a new Gateway` | `First create the new gateway, then select it`
| **NewIPv4 Gateway**
| Default | `☑` Default gateway
| Gateway Name |`WANGW`
| Gateway IPv4 | `192.168.2.5`
| Description | `WAN/VPN Egress Gateway` | *And click `Save` to create the new gateway*
| **Reserved Networks**
| Block private networks and loopback addresses | `☐` | *Uncheck the box*
| Block bogon networks | `☐` | *Uncheck the box*

And click `Save`.

![alt text](https://raw.githubusercontent.com/ahuacate/pfsense-setup/master/images/pfsense_interfaces_03.png)

## 3.00 Add DHCP Servers to OPT1 and OPT2
Because in UniFi we have configured VLAN30 and VLAN40 as `VLAN Only` status we must configure a pfSense DHCP Server for both these VLANs.

### 3.01 Setup DHCP Servers for OPT1 and OPT2
Now using the pfSense web interface `Services` > `DHCP Server` > `OPT1 Tab` or `OPT2 Tab` to open a configuration form, then fill up the necessary fields as follows:

| General Options | OPT 1 Value | OPT2 Value | Notes |
| :---  | :---: | :---: | :---
| Enable | `☑` |  `☑` | *Opt1&2 Check the box*
| BOOTP | `☐` | `☐` | *Disable*
| Deny unknown clients | `☐` | `☐` | *Disable*
| Ignore denied clients | `☐` | `☐` | *Disable*
| Ignore client identifiers | `☐` | `☐` | *Disable*
| Subnet | 192.168.30.0 | 192.168.40.0 |
| Subnet mask |255.255.255.0 | 255.255.255.0 | 
| Available range | 192.168.30.1 - 192.168.30.254 | 192.168.40.1 - 192.168.40.254 |
| Range | `192.168.30.150 - 192.168.30.250` | `192.168.40.150 - 192.168.40.250` |
| **Servers**
| WINS servers | Leave blank
| DNS servers | Leave Blank | Leave Blank | *DNS Server 1-4: Must leave all blank. We are going to use DNS Resolver for DNS tasks. If you add DNS here then pFBlockerNG will not work.*
| **Other Options**
| Gateway | `192.168.30.5` | `192.168.40.5`
| Default Lease time | `86320` | `86320`
| Maximum lease time | `86400` | `86400`

Remember to hit the `Save` button at the bottom of the page.

## 4.00 Setup OpenVPN Client Server
Now this guide presumes you’re subscribing to a VPN service provider. This guide uses ExpressVPN where a standard subscription includes 5x user accounts. So we are going to create 5x OpenVPN clients: 2x vpngate-world-0(x) and 3x vpngate-local-0(x).

Now you will need to fill out many of these boxes with information from your chosen VPN provider and you will need to duplicate it for each server you want to connect with.

It is important also that you give each OpenVPN Client you setup a descriptive name. I create mine like this:

| VPN Client Description Name | Notes
| :---  | :---
| **vpngate-world Name** | *For clients like NZBGet, Deluge where the vpn servers randomly connect in chosen nations*
| vpngate-world-01
| vpngate-world-02
| **vpngate-local Name** | *Main household VPN where the vpn servers connect in the region*
| vpngate-local-01
| vpngate-local-02
| vpngate-local-03

![alt text](https://raw.githubusercontent.com/ahuacate/pfsense-setup/master/images/pfsense_openvpn_01.png)

For the end part of this guide we will be enabling load balancing across two (vpngate-world) and three (vpngate-local) separate VPN servers. Also you can configure only one OpenVPN client or more, it’s completely your choice.

You will need your VPN account server username and password details and have your vpn server provider OVPN configuration file open in a text editor so you can copy various certificate and key details (cut & paste). Note the values for this form will vary between different VPN providers but there should be a tutorial showing your providers pfSense configuration settings on the internet somewhere. 

### 4.01 Add your OpenVPN Client Certificate details
Now using the pfSense web interface `System` > `Cert. Manager` > `CAs` > `Add` to open a configuration form, then fill up the necessary fields as follows:

| Create/Edit CA | Value | Notes
| :---  | :---: | :--- |
| Descriptive name | `ExpressVPN` | *Or whatever your providers name is, ExpressVPN, PIA etc*
| Method | `Import an existing Certificate Authority`
| **Existing Certificate Authority**
| Certificate data | `Insert your key data` | *Open the OpenVPN configuration file that you downloaded and open it with your favorite text editor. Look for the text that is wrapped within the <ca> portion of the file. Copying the entire string from —–BEGIN CERTIFICATE—– to —–END CERTIFICATE—–* |
| Certificate Private Key (optional) | Leave this blank
| Serial for next certificate | Leave this blank

Click `Save`. Stay on this page and click `Certificates` at the top. Click `Add`enter the following:

| Add/Sign a New Certificate | Value | Notes
| :---  | :---: | :--- |
| Descriptive name | `ExpressVPN Cert` | *Or whatever your providers name is, PIA Cert etc*
| **Import Certificate**
| Certificate data | `Insert your key data` | *Open the OpenVPN configuration file that you downloaded and open it with your favorite text editor. Look for the text that is wrapped within the <cert> portion of the file. Copy the entire string from —–BEGIN CERTIFICATE—– to —–END CERTIFICATE—–*
| Private key data | `Insert your key data` | *With your text editor still open, look for the text that is wrapped within the <key> portion of the file. Copy the entire string from —–BEGIN RSA PRIVATE KEY—– to —-END RSA PRIVATE KEY—-*

Click `Save`.

### 4.02 Create your OpenVPN Client Server(s)
Now using the pfSense web interface `VPN` > `OpenVPN` > `Clients Tab` > `Add` to open a configuration form, then fill up the necessary fields as follows (creating one each for `vpngate-world-01`, `vpngate-world-02` and `vpngate-local-01`, `vpngate-local-02`, `vpngate-local-03` ):

| General Information | Value | Notes
| :---  | :---: | :--- |
| Disabled | Leave this box unchecked
|Server Mode | `Peer to Peer (SSL/TLS)`
| Protocol | `UDP on IPv4 only`
| Device mode | `tun - Layer 3 Tunnel Mode`
| Interface | `WAN`
| Local port | Leave blank
| Server host or address : vpngate-world-01 | `netherlands-amsterdam-ca-version-2.expressnetw.com`| *Open the OpenVPN configuration file that you downloaded and open it with your favorite text editor. Look for text that starts with remote, followed by a server name. Copy the server name string into this field (e.g., if you are in London maybe you wnat to use servers outside of your jurisdiction like netherlands-amsterdam-ca-version-2.expressnetw.com, germany-frankfurt-1-ca-version-2.expressnetw.com )*
| Server host or address : vpngate-local-01 | `malaysia-ca-version-2.expressnetw.com` | *Open the OpenVPN configuration file that you downloaded and open it with your favorite text editor. Look for text that starts with remote, followed by a server name. Copy the server name string into this field (e.g., if you are in Hong Kong maybe you wnat to use servers near your region like singapore-jurong-ca-version-2.expressnetw.com, malaysia-ca-version-2.expressnetw.com )*
| Server port | `1195`
| Proxy host or address | Leave blank
| Proxy port | Leave blank
| Proxy authentication… | leave default
| Server host name resolution | leave default (none)
| Description | `vpngate-world-01` or `vpngate-local-01` | *Simply type vpngate-world-0X or vpngate-local-0X for the connection service you are creating*
| **User Authentication Settings**
| Username | `insert your account username`
| Password | `insert your account password`
| Authentication Retry | Leave disabled/default
| **Cryptographic Settings**
| TLS Configuration | `☑ Use a TLS Key` | *Check the box*
| TLS Configuration | `[ ] Automatically generate a TLS Key.` | *Uncheck the box*
| TLS Key | `Insert your key data` | *Open the OpenVPN configuration file that you downloaded and open it with your favorite text editor. Look for text that is wrapped within the <tls-auth> portion of the file. Ignore the “2048 bit OpenVPN static key” entries and start copying from —–BEGIN OpenVPN Static key V1—– to —–END OpenVPN Static key V1—–*
| TLS Key Usage Mode | `TLS Authentication`
| Peer Certificate Authority | `ExpressVPN` | *Select the “ExpressVPN” entry that you created previously in the Cert. Manager steps*
| Client Certificate | `ExpressVPN Cert` | *Select the “ExpressVPN Cert” entry that you created previously in the Cert. Manager steps*
| Encryption Algorithm | `AES-256-CBC` | *Open the OpenVPN configuration file that you downloaded and open it with your favorite text editor. Look for the text cipher. In this example, the OpenVPN configuration is listed as “cipher AES-256-CBC,” so we will select “AES-256-CBC (256-bit key, 128-bit block) from the drop-down*
| Enable NCP | `[ ] Enable Negotiable Cryptographic Parameters` | *Uncheck the box*
| NCP Algorithms | `AES-256-CBC` | Only use/add AES-256-CBC
| Auth digest algorithm | `SHA512 (512-bit)`| *Open the OpenVPN configuration file that you downloaded and open it with your favorite text editor. Look for the text auth followed by the algorithm after. In this example, we saw “auth SHA512,” so we will select “SHA512 (512-bit)” from the dropdown*
| Hardware Crypto | `Intel RDRAND engine - RAND` | *Thats what shows on a Qotom Mini PC Q500G6-S05 which has a AES-NI ready Intel i5-7300U CPU*
| **Tunnel Settings**
| IPv4 Tunnel Network | Leave blank
| IPv6 Tunnel Network | Leave blank
| IPv4 Remote network(s) | Leave blank
| IPv6 Remote network(s) | Leave blank
| Limit outgoing bandwidth | Leave blank
| Compression | `Adaptive LZO Compression`
| Topology | `Leave the default` | *Subnet — One IP address per client in a common subnet*
| Type-of-Service |  [ ] Leave unchecked
| Don't pull routes | `☑`| *Check the box*
| Don't add/remove routes | `☑` | *Check the box*
| **Advanced Configuration**
| Custom options : route-nopull;fast-io;persist-key;persist-tun;pull;comp-lzo;tls-client;verify-x509-name Server name-prefix;remote-cert-tls server;key-direction 1;route-method exe;route-delay 2;tun-mtu 1500;fragment 1300;mssfix 1450;verb 3;sndbuf 524288;rcvbuf 524288
| UDP Fast I/O | `☑ Use fast I/O operations with UDP writes to tun/tap. Experimental.` | *Check the box*
| Send/Receive Buffer | `512`
| Gateway creation  | `IPv4 Only`
| Verbosity level | `3 (recommended)`

Click `Save`.

Then to check whether the connection works navigate `Status` > `OpenVPN` and Status field for vpngate-world-01/02 and vpngate-local-01/02/03 should show `up`. This means you are connected to your provider.

![alt text](https://raw.githubusercontent.com/ahuacate/pfsense-setup/master/images/pfsense_openvpn_02.png)

## 5.00 Setting up a gateway for each OpenVPN Client
Gateways act just like the WAN or LAN does, it’s almost like a physical port on your firewall except in this case it’s for your VPN and not a physical port on the device. Creating a gateway allows us to specify traffic that should utilise the specific VPN Client the gateway is for.

Next we need to add an interface for each new OpenVPN connection and then a Gateway for each interface. 

### 5.01 Add Network Interface Ports
Now using the pfSense web interface `Interfaces` > `Assignments` the configuration form will show FIVE available network ports which can be added:
*  ovpnc1 >> (vpngate-world01)
*  ovpnc2 >> (vpngate-world02)
*  ovpnc3 >> (vpngate-local01)
*  ovpnc4 >> (vpngate-local02)
*  ovpnc5 >> (vpngate-local03)

Now `Add` all and remember to click `Save`. You should create a gateway like this for each OpenVPN Client you created. After clicking `+Add` the interface will be enabled and should receive a generic name such as `OPT3` to `OPT7` (if your created 5x OpenVPN clients). 

### 5.02 Edit the new Interface Ports
Then click on the corresponding `Interface` names one at a time, likely to be `OPT3`, `OPT4`, `OPT5`,`OPT6` and `OPT7`,  and edit the necessary fields as follows:

The first edit will be `OPT3`:

| Interfaces/OPT3 (ovpnc1) | Value | Notes
| :---  | :---: | :--- |
| Enable | `☑` Enable interface | *Check the box*
| Description | `vpngateworld01`
| IPv4/IPv6 Configuration | This interface type does not support manual address configuration on this page.
| MAC Address | `None`
| MTU | Leave blank
| MTU | Leave blank
| MSS | Leave blank
| **Reserved Networks**
| Block private networks and loopback addresses | `☐` | *Uncheck the box*
| Block bogon networks | `☐` | *Uncheck the box*

Click `Save`.

Now edit `OPT4`:

| Interfaces/OPT4 (ovpnc2) | Value | Notes
| :---  | :---: | :--- |
| Enable | `☑` Enable interface | *Check the box*
| Description | `vpngateworld02`
| IPv4/IPv6 Configuration | This interface type does not support manual address configuration on this page.
| MAC Address | `None`
| MTU | Leave blank
| MTU | Leave blank
| MSS | Leave blank
| **Reserved Networks**
| Block private networks and loopback addresses | `☐` | *Uncheck the box*
| Block bogon networks | `☐` | *Uncheck the box*

Click `Save`. 

Now edit `OPT5`:

| Interfaces/OPT5 ovpnc3) | Value | Notes
| :---  | :---: | :--- |
| Enable | `☑` Enable interface | *Check the box*
| Description | `vpngatelocal01`
| IPv4/IPv6 Configuration | This interface type does not support manual address configuration on this page.
| MAC Address | `None`
| MTU | Leave blank
| MTU | Leave blank
| MSS | Leave blank
| **Reserved Networks**
| Block private networks and loopback addresses | `☐` | *Uncheck the box*
| Block bogon networks | `☐` | *Uncheck the box*

Click `Save`.

Now edit `OPT6`:

| Interfaces/OPT6 (ovpnc4) | Value | Notes
| :---  | :---: | :--- |
| Enable | `☑` Enable interface | *Check the box*
| Description | `vpngatelocal02`
| IPv4/IPv6 Configuration | This interface type does not support manual address configuration on this page.
| MAC Address | `None`
| MTU | Leave blank
| MTU | Leave blank
| MSS | Leave blank
| **Reserved Networks**
| Block private networks and loopback addresses | `☐` | *Uncheck the box*
| Block bogon networks | `☐` | *Uncheck the box*

Click `Save`.

Now edit `OPT7`:

| Interfaces/OPT7 (ovpnc5) | Value | Notes
| :---  | :---: | :--- |
| Enable | `☑` Enable interface | *Check the box*
| Description | `vpngatelocal03`
| IPv4/IPv6 Configuration | This interface type does not support manual address configuration on this page.
| MAC Address | `None`
| MTU | Leave blank
| MTU | Leave blank
| MSS | Leave blank
| **Reserved Networks**
| Block private networks and loopback addresses | `☐` | *Uncheck the box*
| Block bogon networks | `☐` | *Uncheck the box*

Click `Save` and `Apply`.

Your `Interfaces` > `Assignments` tab should now look like this:

![alt text](https://raw.githubusercontent.com/ahuacate/pfsense-setup/master/images/pfsense_interfaces_04.png)

### 5.03 Check your Gateway Interface(s)
Your Gateways should be automatically created after completing the above steps. To check go to your pfSense web interface `System` > `Routing` > `Gateways Tab` and it should look like:

![alt text](https://raw.githubusercontent.com/ahuacate/pfsense-setup/master/images/pfsense_gateway_00.png)

## 6.00 Create Gateway Groups
Now this is something very few people setup because they either aren’t aware it can be done or it seems too complicated to setup but it’s actually very easy. You should've already created multiple OpenVPN clients and Gateways and now you simply setup 2 or more (there really is no limit) OpenVPN Clients to utilise within a load balancing group.

The way this load balancing works is round-robin. For example if you load a website you’re not making one single connection to it, you’re actually downloading many different files at once which your browser is then combining together. Such as Images, HTML, CSS, JavaScript and other included but separate files.

It’s a similar situation with BitTorrent. You don’t make a single connection to download a file instead you connect to multiple peers and request different pieces which the BitTorrent client then combines into one final file or folder of files.

These kinds of usage scenarios are thus very appropriate for this kind of load balancing where each new outgoing connection you make chooses a random gateway from your gateway group. It also helps with redundancy so when one OpenVPN connection goes down the other ones will continue to be utilised and pfSense is smart enough to route traffic out over only the gateways that are functioning correctly.

You can even tell pfSense to only make requests through the gateways that have no packet loss and aren’t suffering high latency.

### 6.01 Create VPNGATEWORLD Gateway Group
Here we will create a new Gateway comprising of `VPNGATEWORLD01_VPNV4` and `VPNGATEWORLD02_VPNV4` Gateways.

Now using the pfSense web interface go to `System` > `Routing` > `Gateway Groups` > `+Add` and fill out the necessary fields as follows:

| Edit Gateway Group Entry | Value | Value | Value | Value
| :---  | :--- | :--- | :--- | :--- |
| group Name | `VPNGATEWORLD_GROUP`
| **Gateway Priority** |Gateway | Gateway | Tier | Virtual IP | Description 
| | WANGW | Never | Interface Address | WAN/VPN Egress Gateway
| | VPNGATELOCAL01_VPNV4 | Never | Interface Address | Interface VPNGATELOCAL01_VPNV4 Gateway
| | VPNGATELOCAL02_VPNV4 | Never | Interface Address | Interface VPNGATELOCAL02_VPNV4 Gateway
| | VPNGATELOCAL03_VPNV4 | Never | Interface Address | Interface VPNGATELOCAL03_VPNV4 Gateway
| | VPNGATEWORLD01_VPNV4 | Tier 1 | Interface Address | Interface VPNGATEWORLD01_VPNV4 Gateway
| | VPNGATEWORLD02_VPNV4 | Tier 1 | Interface Address | Interface VPNGATEWORLD02_VPNV4 Gateway
| Trigger Level | `Packet Loss or High Latency`
| Description | VPNGATEWORLD GROUP

And click `Save`.

![alt text](https://raw.githubusercontent.com/ahuacate/pfsense-setup/master/images/pfsense_gatewaygrp_01.png)

### 6.02 Create VPNGATELOCAL Gateway Group
Here we will create a new Gateway comprising of `VPNGATELOCAL01_VPNV4`, `VPNGATELOCAL02_VPNV4` and `VPNGATELOCAL03_VPNV4`Gateways.

Now using the pfSense web interface go to `System` > `Routing` > `Gateway Groups` > `+Add` and fill out the necessary fields as follows:

| Edit Gateway Group Entry | Value | Value | Value | Value
| :---  | :--- | :--- | :--- | :--- |
| group Name | `VPNGATELOCAL_GROUP`
| **Gateway Priority** |Gateway | Gateway | Tier | Virtual IP | Description 
| | WANGW | Never | Interface Address | WAN/VPN Egress Gateway
| | VPNGATELOCAL01_VPNV4 | Tier 1 | Interface Address | Interface VPNGATELOCAL01_VPNV4 Gateway
| | VPNGATELOCAL02_VPNV4 | Tier 1 | Interface Address | Interface VPNGATELOCAL02_VPNV4 Gateway
| | VPNGATELOCAL03_VPNV4 | Tier 1 | Interface Address | Interface VPNGATELOCAL03_VPNV4 Gateway
| | VPNGATEWORLD01_VPNV4 | Never | Interface Address | Interface VPNGATEWORLD01_VPNV4 Gateway
| | VPNGATEWORLD02_VPNV4 | Never | Interface Address | Interface VPNGATEWORLD02_VPNV4 Gateway
| Trigger Level | `Packet Loss or High Latency`
| Description | VPNGATELOCAL GROUP

And click `Save`.

![alt text](https://raw.githubusercontent.com/ahuacate/pfsense-setup/master/images/pfsense_gatewaygrp_02.png)

**IMPORTANT**: At this point you are ready to create the firewall rules. But I would **highly recommend a reboot** here as this was the only thing that made the next few steps work. So do a reboot `Diagnostics` > `Reboot` and perform a `Reboot`.

If you don't things might not work in the steps ahead.

## 7.00 Adding Firewall Aliases
Aliases act as placeholders for real hosts, networks or ports. They can be used to minimize the number of changes that have to be made if a host, network or port changes. The name of an alias can be entered instead of the IP address, network or port in all fields that have a red background. In simple terms --- use them.

### 7.01 Create Firewall Alias - Chromecast IP List
Here you create entries for all your Chromecast and TV device IPs.

But first you MUST assign your Chromecast & TV's IP a static IP address (do it in UniFi) and then enter the static IP address into the `Chromecast_IP_Addresses` alias list.

Now create a new alias `Firewall` > `Aliases` > `IP Tab` > `Add` to open a new configuration form, then fill up the necessary fields as follows and adding your chromecast and TV devices:

| Alias Field | Value | Value |
| :---  | :--- | :--- |
| **Properties**
| Name | `Chromecast_IP_Addresses`
| Description | `List of Chromecast device IPv4 Addresses`
| Type | `Host(s)`
| **Hosts(s)**
| IP ir FQDN | `192.168.20.151` | `Chromecast Living Room`

## 8.00 Adding NAT Rules
Next we need to add the NAT rules to allow for traffic to go out of the VPN encrypted gateway(s), this is done from the `Firewall` > `NAT` > `Outbound Tab`. 

If you have Automatic NAT enabled you want to enable Manual Outbound NAT and click `Save`. Now you will see and be able to edit the NAT Mappings configuration form.

But first you must find any rules that allows the devices you wish to tunnel, with a `Source` value of `192.168.30.0/24` and `192.168.40.0/24` and delete them and click `Save` at the bottom right of the form page. **DO NOT DELETE** the `Mappings` with `Source` values like `127.0.0.0/8, ::1/128, 192.168.1.0/24`!

### 8.01 Create NAT Rule VLAN30 to vpngate-world01/02 - Outbound
Now create new mappings by `Firewall` > `NAT` > `Outbound Tab` > `^Add` (arrow up) to open a new configuration form, then fill up the necessary fields as follows to create two new entries:

| Edit Advanced Outbound NAT Entry | Value | Value
| :---  | :--- | :--- |
| Disabled | `☐` Disable this rule
| Do not NAT | `☐` Enabling this option will .....
| Interface | `VPNGATEWORLD01`
| Address Family | `IPv4`
| Protocol | `any`
| Protocol | `any`
| Source
| | `Network` | `192.168.30.0/24`
| | Port or Range | Leave Blank
| Destination 
| | `Any` | Leave blank
| | Port or Range | Leave Blank
| | `☐` Not
| **Translation**
| Address | `Interface Address`
| Port or Range | Leave blank
| **Misc**
| No XMLRPC Sync | `☐`
| Description | `VLAN30 to vpngate-world01`

And click `Save`.

| Edit Advanced Outbound NAT Entry | Value | Value
| :---  | :--- | :--- |
| Disabled | `☐` Disable this rule
| Do not NAT | `☐` Enabling this option will .....
| Interface | `VPNGATEWORLD02`
| Address Family | `IPv4`
| Protocol | `any`
| Source
| | `Network` | `192.168.30.0/24`
| | Port or Range | Leave Blank
| Destination 
| | `Any` | Leave blank
| | Port or Range | Leave Blank
| | `☐` Not
| **Translation**
| Address | `Interface Address`
| Port or Range | Leave blank
| **Misc**
| No XMLRPC Sync | `☐`
| Description | `VLAN30 to vpngate-world02`

And click `Save`. A example for the `VLAN30 to vpngate-world01` is shown here:

![alt text](https://raw.githubusercontent.com/ahuacate/pfsense-setup/master/images/pfsense_nat_01.png)

### 8.02 Create NAT Rule VLAN40 to vpngate-local - Outbound
Now create new mappings by `Firewall` > `NAT` > `Outband Tab` > `^Add` (arrow up) to open a new configuration form, then fill up the necessary fields as follows to create 3 new entries:

| Edit Advanced Outbound NAT Entry | Value | Value
| :---  | :--- | :--- |
| Disabled | `☐` Disable this rule
| Do not NAT | `☐` Enabling this option will .....
| Interface | `VPNGATELOCAL01`
|Address Family | `IPv4`
| Protocol | `any`
| Source 
| | `Network` | `192.168.40.0/24`
| | Port or Range | Leave Blank
| Destination 
| | `Any` | Leave blank
| | Port or Range | Leave Blank
| | `☐` Not
| **Translation**
| Address | `Interface Address`
| Port or Range | Leave blank
| **Misc**
| No XMLRPC Sync | `☐`
| Description | `VLAN40 to vpngate-local01`

And click `Save`.

| Edit Advanced Outbound NAT Entry | Value | Value
| :---  | :--- | :--- |
| Disabled | `☐` Disable this rule
| Do not NAT | `☐` Enabling this option will .....
| Interface | `VPNGATELOCAL02`
|Address Family | `IPv4`
| Protocol | `any`
| Source 
| | `Network` | `192.168.40.0/24`
| | Port or Range | Leave Blank
| Destination 
| | `Any` | Leave blank
| | Port or Range | Leave Blank
| | `☐` Not
| **Translation**
| Address | `Interface Address`
| Port or Range | Leave blank
| **Misc**
| No XMLRPC Sync | `☐`
| Description | `VLAN40 to vpngate-local02`

And click `Save`.

| Edit Advanced Outbound NAT Entry | Value | Value
| :---  | :--- | :--- |
| Disabled | `☐` Disable this rule
| Do not NAT | `☐` Enabling this option will .....
| Interface | `VPNGATELOCAL03`
|Address Family | `IPv4`
| Protocol | `any`
| Source 
| | `Network` | `192.168.40.0/24`
| | Port or Range | Leave Blank
| Destination 
| | `Any` | Leave blank
| | Port or Range | Leave Blank
| | `☐` Not
| **Translation**
| Address | `Interface Address`
| Port or Range | Leave blank
| **Misc**
| No XMLRPC Sync | `☐`
| Description | `VLAN40 to vpngate-local03`

And click `Save`. A example for the `VLAN40 to vpngate-local01` is shown here:

![alt text](https://raw.githubusercontent.com/ahuacate/pfsense-setup/master/images/pfsense_nat_02.png)

Now your first two mappings for the new gateways show look like this (you may need to rearrange them):

| |Interface|Source|Source Port|Destination|Destination Port|NAT Address|NAT Port|Static Port|Description|
|:--- | :---  | :--- | :--- | :---  | :--- | :--- | :---  | :--- | :--- |
|[]|VPNGATELOCAL01|192.168.40.0/24|*|*|*|VPNGATELOCAL01 address|*|:heavy_check_mark:|VLAN40 to vpngate-local01
|[]|VPNGATELOCAL02|192.168.40.0/24|*|*|*|VPNGATELOCAL02 address|*|:heavy_check_mark:|VLAN40 to vpngate-local02
|[]|VPNGATELOCAL03|192.168.40.0/24|*|*|*|VPNGATELOCAL03 address|*|:heavy_check_mark:|VLAN40 to vpngate-local03
|[]|VPNGATEWORLD01|192.168.30.0/24|*|*|*|VPNGATEWORLD01 address|*|:heavy_check_mark:|VLAN30 to vpngate-world01
|[]|VPNGATEWORLD02|192.168.30.0/24|*|*|*|VPNGATEWORLD02 address|*|:heavy_check_mark:|VLAN30 to vpngate-world02

![alt text](https://raw.githubusercontent.com/ahuacate/pfsense-setup/master/images/pfsense_nat_03.png)


## 9.00 Adding Firewall Rules
This is simple because we are going to send all the traffic in a subnet(s) (VLAN30 >> VPNGATEWORLD_GROUP / VLAN40 >> VPNGATELOCAL_GROUP ) through the openVPN tunnels. 

Remember Firewall rules work top down so best set `Allow Rules` above and `Block Rules` below.

### 9.01 Allow Rule for OPT1 - VPNGATEWORLD_GROUP
Go to  `Firewall` > `Rules` > `OPT1 tab` and `^Add` a new rule:

| Edit Firewall Rule / OPT1 | Value | Notes|
| :---  | :--- | :--- |
| Action | `Pass`
| Disabled | `☐` disable this rule
| Interface | `OPT1`
| Addresss Family | `IPv4`
| Protocol | `Any`
| **Source**
| Source 
| | `☐` Invert match. 
| | `any` 
| | Source Address - leave blank
| **Destination**
| Destination 
| | `☐` Invert match.
| | `any` 
| | Destination Address - leave blank
| **Extra Options**
| Log | `☐` Log packets that are handled by this rule
| Description | `VLAN30 Traffic to vpngateworld-group`
| Advanced Options | Click `Display Advanced` | *This is a important step. You only want to edit one value in `Advanced`!!!!*
| **Advanced Options**
| Gateway | `VPNGATEWORLD_GROUP-VPNGATEWORLD GROUP` | *MUST Change to this gateway!*

Click `Save`.

### 9.02 Allow Rule for OPT2 - VPNGATELOCAL_GROUP
Go to  `Firewall` > `Rules` > `OPT2 tab` and `Add` a new rule:

| Edit Firewall Rule / OPT2 | Value | Notes|
| :---  | :--- | :--- |
| Action | `Pass`
| Disabled | `☐` disable this rule
| Interface | `OPT2`
| Addresss Family | `IPv4`
| Protocol | `Any`
| **Source**
| Source 
| | `☐` Invert match. 
| | `any` 
| | Source Address - leave blank
| **Destination**
| Destination 
| | `☐` Invert match.
| | `any` 
| | Destination Address - leave blank
| **Extra Options**
| Log | `☐` Log packets that are handled by this rule
| Description | `VLAN40 Traffic to vpngatelocal-group`
| Advanced Options | Click `Display Advanced` | *This is a important step. You only want to edit one value in `Advanced`!!!!*
| **Advanced Options**
| Gateway | `VPNGATELOCAL_GROUP-VPNGATELOCAL GROUP` | *MUST Change to this gateway!*

Click `Save` and `Apply`.

The above two rules will send all traffic on each interface into the respective VPN tunnel. You must ensure the ‘gateway’ setting option is set to your VPN gateway and that this rule is above any other rule that allows hosts to go out to the internet. pfSense needs to be able to catch this rule before any others.

### 9.03 Allow OPT2 access to Chromecast and TV devices - vpngate-local
By default VLAN40 (vpngate-local) has no access to any other other VLAN. For convenience we allow certain VLAN40 ports access to our Chromecast Alias IP address.

Go to  `Firewall` > `Rules` > `OPT2 tab` and `^ Add (arrow up)` a new rule:

| Firewall Rule / OPT2 | Value | Notes
| :---  | :--- | :---
| **Edit Firewall Rule **
| Action | `Pass`
| Disabled | `☐` disable this rule
| Interface | `OPT2`
| Addresss Family | `IPv4`
| Protocol | `TCP`
| **Source**
| Source 
| | `☐` Invert match. 
| | `OPT2 net` 
| | Source Address - leave blank
| **Destination**
| Destination 
| | `☐` Invert match.
| | `Single host or alias` 
| | `Chromecast_IP_Addresses` | *Here we use the Firewall alias `Chromecast_IP_Addresses` we created before*
| Destination Port Range
| | From `Other`
| | Custom `8008`
| | To `Other`
| | Custom `8009`
| **Extra Options**
| Log | `☐` Log packets that are handled by this rule
| Description | `Allow OPT2 (VPNGate-local) Port 8008-8009 to Chromecast`
| Advanced Options | Leave Default

Click `Save`.

Now we need to create another rule. Go to  `Firewall` > `Rules` > `OPT2 tab` and `^ Add (arrow up)` a new rule:

| Firewall Rule / OPT2 | Value | Notes
| :---  | :--- | :---
| **Edit Firewall Rule **
| Action | `Pass`
| Disabled | `☐` disable this rule
| Interface | `OPT2`
| Addresss Family | `IPv4`
| Protocol | `UDP`
| **Source**
| Source 
| | `☐` Invert match. 
| | `OPT2 net` 
| | Source Address - leave blank
| **Destination**
| Destination 
| | `☐` Invert match.
| | `Single host or alias` 
| | `224.0.0.251` | 
| Destination Port Range
| | From `Other`
| | Custom `1900`
| | To `Other`
| | Custom `1900`
| **Extra Options**
| Log | `☐` Log packets that are handled by this rule
| Description | `Allow OPT2 (VPNGate-local) Port 1900 to Chromecast`
| Advanced Options | Leave Default

Click `Save`.

Now we need to create another rule. Go to  `Firewall` > `Rules` > `^ OPT2 tab` and `Add (arrow up)` a new rule:

| Firewall Rule / OPT2 | Value | Notes
| :---  | :--- | :---
| **Edit Firewall Rule **
| Action | `Pass`
| Disabled | `☐` disable this rule
| Interface | `OPT2`
| Addresss Family | `IPv4`
| Protocol | `UDP`
| **Source**
| Source 
| | `☐` Invert match. 
| | `OPT2 net` 
| | Source Address - leave blank
| **Destination**
| Destination 
| | `☐` Invert match.
| | `Single host or alias` 
| | `224.0.0.251` | 
| Destination Port Range
| | From `Other`
| | Custom `5353`
| | To `Other`
| | Custom `5353`
| **Extra Options**
| Log | `☐` Log packets that are handled by this rule
| Description | `Allow OPT2 (VPNGate-local) Port 5353 to Chromecast`
| Advanced Options | Leave Default

Click `Save` and `Apply`.

Your `Firewall` > `Rules` > `OPT2 tab` should now look like:

![alt text](https://raw.githubusercontent.com/ahuacate/pfsense-setup/master/images/pfsense_rules_00.png)

### 9.04 Allow WAN to VLAN30 devices (NZBGet & Deluge) - vpngate-world
NZBGet and Deluge are on the VLAN30 vpngate-world network. We need to allow access from other VLANs to VLAN30.

Go to  `Firewall` > `Rules` > `WAN tab` and `^Add` two new rules:

| Firewall Rule / WAN | Value | Notes
| :---  | :--- | :---
| **Edit Firewall Rule **
| Action | `Pass`
| Disabled | `☐` disable this rule
| Interface | `WAN`
| Addresss Family | `IPv4`
| Protocol | `TCP`
| **Source**
| Source 
| | `☐` Invert match. 
| | `any` 
| | Source Address - leave blank
| **Destination**
| Destination 
| | `☐` Invert match.
| | `Single host or alias` 
| | `192.168.30.112` | *Change to your VPN providers DNS*
| Destination Port Range
| | From `Other`
| | Custom `6789`
| | To `Other`
| | Custom `6789`
| **Extra Options**
| Log | `☐` Log packets that are handled by this rule
| Description | `Allow WAN net to VLAN30 NZBGet`
| Advanced Options | Leave Default

Click `Save`.

| Firewall Rule / WAN | Value | Notes
| :---  | :--- | :---
| **Edit Firewall Rule **
| Action | `Pass`
| Disabled | `☐` disable this rule
| Interface | `WAN`
| Addresss Family | `IPv4`
| Protocol | `TCP`
| **Source**
| Source 
| | `☐` Invert match. 
| | `any` 
| | Source Address - leave blank
| **Destination**
| Destination 
| | `☐` Invert match.
| | `Single host or alias` 
| | `192.168.30.113` | *Change to your VPN providers DNS*
| Destination Port Range
| | From `Other`
| | Custom `8112`
| | To `Other`
| | Custom `8112`
| **Extra Options**
| Log | `☐` Log packets that are handled by this rule
| Description | `Allow WAN net to VLAN30 Deluge`
| Advanced Options | Leave Default

Click `Save` and `Apply`.

![alt text](https://raw.githubusercontent.com/ahuacate/pfsense-setup/master/images/pfsense_rules_03.png)

### 9.05 DNS Allow and Block Rules on OPT1 - vpngate-world
When your computer needs to know an IP Address of a host it will use a DNS server and by default it will use your internet service providers or the DNS resolver built into pfSense. This is bad when using a VPN because performing a DNS lookup can reveal your origin IP Address.

To fix this problem we’re going to create a basic firewall rule which will pass traffic destined for UDP port 53 (the DNS server port) on each VLAN to its own VLAN DNS server which will be operated by your VPN provider. Next we add a rule where if the traffic destined for UDP port 53 is'nt on its own VLAN we block it.

You need both rules (Pass & Block) for pfBlockerNG addon to work with DNSBL lists.

Go to  `Firewall` > `Rules` > `OPT1 tab` and `^ Add (arrow up)` a new rule:

| Firewall Rule / OPT1 | Value | Notes
| :---  | :--- | :---
| **Edit Firewall Rule **
| Action | `Block`
| Disabled | `☐` disable this rule
| Interface | `OPT1`
| Addresss Family | `IPv4`
| Protocol | `UDP`
| **Source**
| Source 
| | `☐` Invert match. 
| | `OPT1 net` 
| | Source Address - leave blank
| **Destination**
| Destination 
| | `☐` Invert match.
| | `any` 
| | Destination Address - Leave Blank
| Destination Port Range
| | From `DNS (53)`
| | Custom `Leave Blank`
| | To `DNS (53)`
| | Custom `Leave Blank`
| **Extra Options**
| Log | `☐` Log packets that are handled by this rule
| Description | `Block OPT1 all other DNS`
| Advanced Options | Leave Default

Click `Save` and `Apply`.

Go to  `Firewall` > `Rules` > `OPT1 tab` and `^ Add (arrow up)` a new rule:

| Firewall Rule / OPT1 | Value | Notes
| :---  | :--- | :---
| **Edit Firewall Rule **
| Action | `Pass`
| Disabled | `☐` disable this rule
| Interface | `OPT1`
| Addresss Family | `IPv4`
| Protocol | `UDP`
| **Source**
| Source 
| | `☐` Invert match. 
| | `any` 
| | Source Address - leave blank
| **Destination**
| Destination 
| | `☐` Invert match.
| | `OPT1 net` 
| | Destination Address - Leave Blank
| Destination Port Range
| | From `DNS (53)`
| | Custom `Leave Blank`
| | To `DNS (53)`
| | Custom `Leave Blank`
| **Extra Options**
| Log | `☐` Log packets that are handled by this rule
| Description | `Allow OPT1 DNS to pfSense Resolver`
| Advanced Options | Leave Default

Click `Save` and `Apply`.

![alt text](https://raw.githubusercontent.com/ahuacate/pfsense-setup/master/images/pfsense_rules_01.png)

### 9.06 DNS Allow and Block Rules on OPT2 - vpngate-local
When your computer needs to know an IP Address of a host it will use a DNS server and by default it will use your internet service providers or the DNS resolver built into pfSense. This is bad when using a VPN because performing a DNS lookup can reveal your origin IP Address.

To fix this problem we’re going to create a basic firewall rule which will pass traffic destined for UDP port 53 (the DNS server port) on each VLAN to its own VLAN DNS server which will be operated by your VPN provider. Next we add a rule where if the traffic destined for UDP port 53 is'nt on its own VLAN we block it.

You need both rules (Pass & Block) for pfBlockerNG addon to work with DNSBL lists.

Go to  `Firewall` > `Rules` > `OPT2 tab` and `^ Add (arrow up)` a new rule:

| Firewall Rule / OPT2 | Value | Notes
| :---  | :--- | :---
| **Edit Firewall Rule **
| Action | `Block`
| Disabled | `☐` disable this rule
| Interface | `OPT2`
| Addresss Family | `IPv4`
| Protocol | `UDP`
| **Source**
| Source 
| | `☐` Invert match. 
| | `OPT2 net` 
| | Source Address - leave blank
| **Destination**
| Destination 
| | `☐` Invert match.
| | `any` 
| | Destination Address - Leave Blank
| Destination Port Range
| | From `DNS (53)`
| | Custom `Leave Blank`
| | To `DNS (53)`
| | Custom `Leave Blank`
| **Extra Options**
| Log | `☐` Log packets that are handled by this rule
| Description | `Block OPT2 all other DNS`
| Advanced Options | Leave Default

Click `Save` and `Apply`.

Go to  `Firewall` > `Rules` > `OPT2 tab` and `^ Add (arrow up)` a new rule:

| Firewall Rule / OPT1 | Value | Notes
| :---  | :--- | :---
| **Edit Firewall Rule **
| Action | `Pass`
| Disabled | `☐` disable this rule
| Interface | `OPT2`
| Addresss Family | `IPv4`
| Protocol | `UDP`
| **Source**
| Source 
| | `☐` Invert match. 
| | `any` 
| | Source Address - leave blank
| **Destination**
| Destination 
| | `☐` Invert match.
| | `OPT2 net` 
| | Destination Address - Leave Blank
| Destination Port Range
| | From `DNS (53)`
| | Custom `Leave Blank`
| | To `DNS (53)`
| | Custom `Leave Blank`
| **Extra Options**
| Log | `☐` Log packets that are handled by this rule
| Description | `Allow OPT2 DNS to pfSense Resolver`
| Advanced Options | Leave Default

Click `Save` and `Apply`.

![alt text](https://raw.githubusercontent.com/ahuacate/pfsense-setup/master/images/pfsense_rules_02.png)

## 10.00 Setup pfSense DNS
Here will setup two DNS services.

### 10.01 Set Up DNS Resolver
To configure the pfSense DNS resolver navigate to `Services` > `DNS Resolver` and on the tab `General Settings` fill up the necessary fields as follows:

| DNS Resolver Settings | Value | Notes
| :---  | :--- | :--- 
| **General Settings Tab**
| **General DNS Resolver Options**
| Enable | `☑ Enable DNS resolver` | 
| Listen Port | Leave Default | 
| Enable SSL/TLS Service | `☐` | 
| SSL/TLS Certificate | Leave Default | 
| SSL/TLS Listen Port | Leave Default | *Should be 853 by default*
| Network Interfaces | `LAN` | *Select ONLY LAN, OPT1, OPT2 and Localhost - Use the Ctrl key to toggle selection*
||`OPT1`
||`OPT2`
||`Localhost`
| Outgoing Network Interfaces | `VPNGATEWORLD01` |*Select ONLY VPNGATEWORLD01,VPNGATEWORLD02 and VPNGATELOCAL01, VPNGATELOCAL02, VPNGATELOCAL03 - Use the Ctrl key to toggle selection*
|| `VPNGATEWORLD02`
|| `VPNGATELOCAL01`
|| `VPNGATELOCAL02`
|| `VPNGATELOCAL03`
| System Domain Local Zone Type | `Static` | 
| DNSSEC | `☑ Enable DNSSEC Support` | 
| DNS Query Forwarding | `☐` Enable Forwarding Mode | *Uncheck*
|| `[]` Use SSL/TLS for outgoing DNS Queries to Forwarding Servers | *Uncheck*
| DHCP Registration | `☐` Register DHCP leases in the DNS Resolver | *Uncheck*
| Static DHCP | `☐` Register DHCP static mappings in the DNS Resolver | *Uncheck*
| OpenVPN Clients | `☑` Register connected OpenVPN clients in the DNS Resolver | *Check*
| Display Custom Options | Click `Display Custom Options` | 
| Custom options  | `server:include: /var/unbound/pfb_dnsbl.*conf`
| ** Advanced Settings Tab**
| **Advanced Pricvacy Options**
| Hide Identity | `☑` id.server and hostname.bind queries are refused
| Hide Version | `☑` version.server and version.bind queries are refused
| Query Name Minimization | `☐` Send minimum amount of QNAME/QTYPE information to upstream servers to enhance privacy
| Strict Query Name Minimization | `☐` Do not fall-back to sending full QNAME to potentially broken DNS servers
| **Advanced Resolver Options**
| Prefetch Support | `☐` Message cache elements are prefetched before they expire to help keep the cache up to date
| Prefetch DNS Key Support | `☐` DNSKEYs are fetched earlier in the validation process when a Delegation signer is encountered
| Harden DNSSEC Data | `☑` DNSSEC data is required for trust-anchored zones.

And click `Save`.

### 10.02 Set Up General DNS
Navigate to `System` > `General Settings` and fill up the necessary fields as follows:

| General Setup | Value | Value | Notes
| :---  | :--- | :--- | :---
| **System**
| Hostname | `pfSense`
| Domain | `localdomain`
| **DNS Server Settings**
| DNS Servers | `1.1.1.1` | DNS Hostname - Leave Default | `none` | *Must choose none*
| DNS Servers | `1.0.0.1` | DNS Hostname - Leave Default | `none` | *Must choose none*
| DNS Server Override | `☐` Allow DNS server list to be overridden by DHCP/PPP on WAN
| Disable DNS Forwarder | `☑` Do not use the DNS Forwarder/DNS Resolver as a DNS server for the firewall
| **Localisation**
| Timezone | Select your region
| Timeservers | 0.pfsense.pool.ntp.org
| Language | Select your language

And click `Save`. Your pfSense appliance is now configured for DNS servers.

## 11.00 Install Avahi for mdns
The Avahi package used in pfSense software is a system which facilitates service discovery on a local network. This means that a laptop or computer may be connected into a network and instantly be able to view other people to chat with, locate chromecast devices and find printers to print to or find files being shared.

### 11.01 Install Avahi Package
In the pfSense WebGUI go to System > Package Manager > Available Packages and type ‘avahi’ into the search criteria and then click Search.

Make sure you click `+ Install` and then Confirm on the next page. Installation may take a short while as it downloads and updates certain packages.

### 11.02 Setup Avahi
When choosing your interfaces you must have at least two interface selected: one interface to listen on (i.e LAN) for mdns packets and another pfSense interfaces to repeat the mdns packets to (i.e OPT2).

After installation of Avahi navigate to `Services` > `Avahi` and fill up the necessary fields as follows:

| Avahi Setup | Value | Notes
| :---  | :--- | :--- 
| **General Settings**
| Enable | `pfSense`
| Interface Action | `Allow Interfaces`
| Interfaces 
| | `LAN` | *Avahi listens in this interface for mdns packets*
| | `OPT2`| *Avahi repeats mdns packets to VLAN40 or VPNGATE-LOCAL network*
| Disable IPv4 | `☐` Disable support for IPv4
| Disable IPv6 | `☐` Disable support for IPv6
| Enable Reflection | `☑` Repeat mdns packets across subnets | *Must enable!*
| **Publishing**
| Enable publishing | `☐` Enable publishing of information about the pfSense host

And click `Save`.

![alt text](https://raw.githubusercontent.com/ahuacate/pfsense-setup/master/images/pfsense_ntp_01.png)

## 12.00 Setup NTP
This is easy. Navigate to `Services` > `NTP` >`Settings` and fill up the necessary fields as follows:

![alt text](https://raw.githubusercontent.com/ahuacate/pfsense-setup/master/images/pfsense_avahi_01.png)

## 13.00 Finish Up
After all the above is in place head over to `Diagnostics` > `States` > `Reset States Tab` > and tick `Reset the firewall state table` click `Reset`. After doing any firewall changes that involve a gateway change its best doing a state reset before checking if anything has worked (odds are it will not work if you dont). PfSense WebGUI may hang for period but dont despair because it will return in a few seconds for routing to come back and up to a minute, don’t panic.

And finally navigate to `Diagnostics` > `Reboot` and reboot your pfSense machine.

Once you’re done head over to any client PC on the network or mobile on the WiFi SSID on either `vpngate-world` VLAN30 or `vpngate-local` VLAN40 networks and use IP checker to if its all working https://wtfismyip.com with NO DNS Leaks https://www.dnsleaktest.com/ (use the extended option on dnsleaktest).

Success! (hopefully)

## 14.00 Create a pfSense Backup
If all is working its best to make a backup of your pfsense configuration. Also if you experiment around a lot, it’s an easy way to restore back to a working configuration. Also, do a backup each and every time before upgrading to a newer version of your firewall or pfSense OS. So in the event you have to rebuild pfSense you can skip Steps 7.0 onwards by using the backup restore feature which will save you a lot of time.

On your pfSense WebGUI navigate to `Diagnostics` > `Backup & Restore` then fill up the necessary fields as follows:

| Backup Configuration | Value | Notes
| :---  | :--- | :--- |
| Backup area | `All` | *Select All*
| Skip Packages | `☐` Do not backup package information | *Uncheck the box*
| Skip RRD data | `☑` Do not backup RRD data (NOTE: RRD Data can consume 4+ megabytes of config.xml space!)| *Check the box. RRD Data are your Graphs. Like the traffic Graph for example. I do not back them up because I do not need them*
| Encyption | `☐` Encrypt this configuration file. | *If you check this box a password value box will appear. Dont forget your password otherwise you are truly stuffed if you need to perform a restore*
| Password (optional box) | `xxxxxxxx` | *See above*

And then click the `Download configuration as XML` and `Save` the backup XML file to your NAS or a secure location. Note: If you are using the WebGUI on a Win10 PC the XML backup file will be saved in your users `Downloads` folder where you can then copy/move the file to a safer location. You should have a backup folder share on your NAS so why not store the XML file there `backup/pfsense/config-pfSense.localdomain-2019xxxxxxxxxx.xml`

---

## 00.00 Patches and Fixes

### 00.01 pfSense – disable firewall with pfctl -d
If for whatever reason you have lost access to the pfSense web management console then go to the Proxmox web interface `typhoon-01` > `251 (pfsense)` > `>_ Console` and `Enter an option` numerical `8` to open a shell.

Then type and execute `pfctl -d` where the -d will temporally disable the firewall (you should see the confirmation in the shell `pf disabled`, where pf is the packet filter = FIREWALL)

Now you can log into the WAN side IP address (192.168.2.1) and govern the pfsense again to fix the problem causing pfSense web management console to cease working on 192.168.1.253.
