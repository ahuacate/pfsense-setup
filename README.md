# pfSense - Setup
This recipe is for setting up pfSense. The following assumes your are familiar with pfSEnse and its preference menus and configuration options.

Network prerequisites are:
- [x] Layer 2 Network Switches
- [x] Network Gateway is `192.168.1.5`
- [x] Network DNS server is `192.168.1.5` (Note: your Gateway hardware should enable you to a configure DNS server(s), like a UniFi USG Gateway, so set the following: primary DNS `192.168.1.254` which will be your PiHole server IP address; and, secondary DNS `1.1.1.1` which is a backup Cloudfare DNS server in the event your PiHole server 192.168.1.254 fails or os down)
- [x] Network DHCP server is `192.168.1.5`

Other Prerequisites are:
- [x] Proxmox node fully configured as per [PROXMOX-NODE BUILDING](https://github.com/ahuacate/proxmox-node/blob/master/README.md#proxmox-node-building)
- [x] pfSense is fully configured on typhoon-01 including both OpenVPN Gateways VPNGATE-LOCAL and VPNGATE-WORLD.


Tasks to be performed are:

- [ ] 00.00 Patches & Fixes


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
| Block bogon networks | `[]` | *Uncheck the box*

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
| Block bogon networks | `[]` | *Uncheck the box*

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
| Block private networks and loopback addresses | `[]` | *Uncheck the box*
| Block bogon networks | `[]` | *Uncheck the box*

And click `Save`.

![alt text](https://raw.githubusercontent.com/ahuacate/pfsense-setup/master/images/pfsense_interfaces_03.png)

## 3.00 Add DHCP Servers to OPT1 and OPT2
Because in UniFi we have configured VLAN30 and VLAN40 as `VLAN Only` status we must configure a pfSense DHCP Server for both these VLANs.

### 3.01 Setup DHCP Servers for OPT1 and OPT2
Now using the pfSense web interface `Services` > `DHCP Server` > `OPT1 Tab` or `OPT2 Tab` to open a configuration form, then fill up the necessary fields as follows:

| General Options | OPT 1 Value | OPT2 Value | Notes |
| :---  | :---: | :---: | :---
| Enable | `☑` |  `☑` | *Opt1&2 Check the box*
| BOOTP | `[ ]` | `[ ]` | *Disable*
| Deny unknown clients | `[ ]` | `[ ]` | *Disable*
| Ignore denied clients | `[ ]` | `[ ]` | *Disable*
| Ignore client identifiers | `[ ]` | `[ ]` | *Disable*
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
Here we going to create OpenVPN clients vpngate-world and vpngate-local. You will need your VPN account server username and password details and have your vpn server provider OVPN configuration file open in a text editor so you can copy various certificate and key details (cut & paste). Note the values for this form will vary between different VPN providers but there should be a tutorial showing your providers pfSense configuration settings on the internet somewhere. 

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

### 4.02 Create your OpenVPN Client Server
Now using the pfSense web interface `VPN` > `OpenVPN` > `Clients Tab` > `Add` to open a configuration form, then fill up the necessary fields as follows (creating one each for `vpngate-world` and `vpngate-local`):

| General Information | Value | Notes
| :---  | :---: | :--- |
| Disabled | Leave this box unchecked
|Server Mode | `Peer to Peer (SSL/TLS)`
| Protocol | `UDP on IPv4 only`
| Device mode | `tun - Layer 3 Tunnel Mode`
| Interface | `WAN`
| Local port | Leave blank
| Server host or address : vpngate-world | `netherlands-amsterdam-ca-version-2.expressnetw.com`| *Open the OpenVPN configuration file that you downloaded and open it with your favorite text editor. Look for text that starts with remote, followed by a server name. Copy the server name string into this field (e.g., if you are in USA maybe you wnat to use servers outside of your jurisdiction like netherlands-amsterdam-ca-version-2.expressnetw.com )*
| Server host or address : vpngate-local | `thailand-ca-version-2.expressnetw.com` | *Open the OpenVPN configuration file that you downloaded and open it with your favorite text editor. Look for text that starts with remote, followed by a server name. Copy the server name string into this field (e.g., if you are in South East Asia maybe you wnat to use servers near your region like thailand-ca-version-2.expressnetw.com )*
| Server port | `1195`
| Proxy host or address | Leave blank
| Proxy port | Leave blank
| Proxy authentication… | leave default
| Server host name resolution | leave default (none)
| Description | `vpngate-world` or `vpngate-local` | *Simply type vpngate-world or vpngate-local for the connection service you are creating*
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
| Custom options : vpngate-world| `route-nopull;fast-io;persist-key;persist-tun;remote sweden-2-ca-version-2.expressnetw.com 1195;remote singapore-cbd-ca-version-2.expressnetw.com 1195;remote germany-frankfurt-1-ca-version-2.expressnetw.com 1195;remote-random;pull;comp-lzo;tls-client;verify-x509-name Server name-prefix;remote-cert-tls server;key-direction 1;route-method exe;route-delay 2;tun-mtu 1500;fragment 1300;mssfix 1450;verb 3;sndbuf 524288;rcvbuf 524288` | *These options are derived from the OpenVPN configuration you have been referencing. We will be pulling out all custom options that we have not used previously. Note, the addition of `route-nopull`. Also, for redundancy we add other remote servers in the event the primary server fails*
| Custom options : vpngate-local| `route-nopull;fast-io;persist-key;persist-tun;remote-random;pull;comp-lzo;tls-client;verify-x509-name Server name-prefix;remote-cert-tls server;key-direction 1;route-method exe;route-delay 2;tun-mtu 1500;fragment 1300;mssfix 1450;verb 3;sndbuf 524288;rcvbuf 524288` | *These options are derived from the OpenVPN configuration you have been referencing. We will be pulling out all custom options that we have not used previously. Note, the addition of `route-nopull`.*
| UDP Fast I/O | `☑ Use fast I/O operations with UDP writes to tun/tap. Experimental.` | *Check the box*
| Send/Receive Buffer | `512`
| Gateway creation  | `IPv4 Only`
| Verbosity level | `3 (recommended)`

Click `Save`.

Then to check whether the connection works navigate `Status` > `OpenVPN` and Status field for vpngate-world and vpngate-local should show `up`. This means you are connected to your provider.

## 5.00 Create two new Gateways
Gateways act just like the WAN or LAN does, it’s almost like a physical port on your firewall except in this case it’s for your VPN and not a physical port on the device. Creating a gateway allows us to specify traffic that should utilise the specific VPN Client the gateway is for.

Next we need to add an interface for each new OpenVPN connection and then a Gateway for each interface. 

### 5.01 Add Network Interface Ports
Now using the pfSense web interface `Interfaces` > `Assignments` the configuration form will show two available network ports which can be added, ovpnc1 (vpngate-world) and ovpnc2 (vpngate-local). Now `Add` both and remember to click `Save`.

### 5.02 Edit the new Interface Ports
Then click on the corresponding `Interface` names one at a time, likely to be `OPT3` and `OPT4`,  and edit the necessary fields as follows (editing both OPT3 and OPT4):

The first edit will be `OPT3`:

| Interfaces/OPT3 (ovpnc1) | Value | Notes
| :---  | :---: | :--- |
| Enable | `☑` Enable interface | *Check the box*
| Description | `vpngateworld`
| IPv4/IPv6 Configuration | This interface type does not support manual address configuration on this page.
| MAC Address | `None`
| MTU | Leave blank
| MTU | Leave blank
| MSS | Leave blank
| **Reserved Networks**
| Block private networks and loopback addresses | `[]` | *Uncheck the box*
| Block bogon networks | `[]` | *Uncheck the box*

Click `Save`.

Now edit `OPT4`:

| Interfaces/OPT4 (ovpnc2) | Value | Notes
| :---  | :---: | :--- |
| Enable | `☑` Enable interface | *Check the box*
| Description | `vpngatelocal`
| IPv4/IPv6 Configuration | This interface type does not support manual address configuration on this page.
| MAC Address | `None`
| MTU | Leave blank
| MTU | Leave blank
| MSS | Leave blank
| **Reserved Networks**
| Block private networks and loopback addresses | `[]` | *Uncheck the box*
| Block bogon networks | `[]` | *Uncheck the box*

Click `Save`. 

Your `Interfaces` > `Assignments` tab should now look like this:

![alt text](https://raw.githubusercontent.com/ahuacate/pfsense-setup/master/images/pfsense_interfaces_04.png)

### 5.03 Setting up a Gateway for vpngate-world
Now using the pfSense web interface go to `System` > `Routing` > `Gateways` and edit the necessary fields as follows:

| Edit Gateway | Value | Notes
| :---  | :---: | :--- |
| Disabled | `[]` Disable this gateway 
| Interface | `VPNGATEWORLD`
| Address Family | `IPv4`
| Name | `VPNGATEWORLD_VPNV4`
| Gateway | dynamic
| Gateway Monitoring | `[]` Disable Gateway Monitoring 
| Gateway Action | `[]` Disable Gateway Monitoring
| Monitor IP | `85.203.37.1` | *Change to your VPN providers DNS*
| Force State | `[]` Mark Gateway as Down
| Description | `Interface VPNGATEWORLD_VPNV4 Gateway`

And click `Save`.

### 5.04 Setting up a Gateway for vpngate-local
Now using the pfSense web interface go to `System` > `Routing` > `Gateways` and edit the necessary fields as follows:

| Edit Gateway | Value | Notes
| :---  | :---: | :--- |
| Disabled | `[]` Disable this gateway 
| Interface | `VPNGATELOCAL`
| Address Family | `IPv4`
| Name | `VPNGATELOCAL_VPNV4`
| Gateway | dynamic
| Gateway Monitoring | `[]` Disable Gateway Monitoring 
| Gateway Action | `[]` Disable Gateway Monitoring
| Monitor IP | `85.203.37.1` | *Change to your VPN providers DNS*
| Force State | `[]` Mark Gateway as Down
| Description | `Interface VPNGATELOCAL_VPNV4 Gateway`

And click `Save`. Your pfSense web interface go to `System` > `Routing` > `Gateways Tab` should look like:

![alt text](https://raw.githubusercontent.com/ahuacate/pfsense-setup/master/images/pfsense_gateway_00.png)

**IMPORTANT**: At this point you are ready to create the firewall rules. But I would **highly recommend a reboot** here as this was the only thing that made the next few steps work. So do a reboot `Diagnostics` > `Reboot` and perform a `Reboot`.

If you don't things might not work in the steps ahead.

## 6.00 Adding Firewall Aliases
Aliases act as placeholders for real hosts, networks or ports. They can be used to minimize the number of changes that have to be made if a host, network or port changes. The name of an alias can be entered instead of the IP address, network or port in all fields that have a red background. In simple terms --- use them.

### 6.01 Create Firewall Alias - Chromecast IP List
Here you list all your Chromecast and TV device IPs. So when you add a Chromecast IP assign it a static IP and enter it into the `Chromecast_IP_Addresses` alias list.

Now create a new alias `Firewall` > `Aliases` > `IP Tab` > `Add` to open a new configuration form, then fill up the necessary fields as follows and adding your chromecast and TV devices:

| Alias Field | Value | Value |
| :---  | :--- | :--- |
| **Properties**
| Name | `Chromecast_IP_Addresses`
| Description | `List of Chromecast device IPv4 Addresses`
| Type | `Host(s)`
| **Hosts(s)**
| IP ir FQDN | `192.168.20.151` | `Chromecast Living Room`

## 7.00 Adding NAT Rules - Outbound
Next we need to add the NAT rules to allow for traffic to go out of the VPN encrypted gateway(s), this is done from the `Firewall` > `NAT` > `Outbound Tab`. 

If you have Automatic NAT enabled you want to enable Manual Outbound NAT and click `Save`. Now you will see and be able to edit the NAT Mappings configuration form.

But first you must find any rules that allows the devices you wish to tunnel, with a `Source` value of `192.168.30.0/24` and `192.168.40.0/24` and delete them and click `Save` at the bottom right of the form page. **DO NOT DELETE** the `Mappings` with `Source` values like `127.0.0.0/8, ::1/128, 192.168.1.0/24`!

### 7.01 Create NAT Rule VLAN30 to vpngate-world
Now create new mappings by `Firewall` > `NAT` > `Outband Tab` > `Add` to open a new configuration form, then fill up the necessary fields as follows:

| Edit Advanced Outbound NAT Entry | Value | Value
| :---  | :--- | :--- |
| Disabled | `[]` Disable this rule
| Do not NAT | `[]` Enabling this option will .....
| Interface | `VPNGATEWORLD`
| Address Family | `IPv4+IPv6`
| Protocol | `any`
| Source | `Network` | `192.168.30.0/24`
| Destination | `Any` | Leave blank
| **Translation**
| Address | `Interface Address`
| Port or Range | Leave blank
| **Misc**
| No XMLRPC Sync | `[]`
| Description | `VLAN30 to vpngate-world`

And click `Save`.

![alt text](https://raw.githubusercontent.com/ahuacate/pfsense-setup/master/images/pfsense_nat_01.png)

### 7.02 Create NAT Rule VLAN40 to vpngate-local
Now create new mappings by `Firewall` > `NAT` > `Outband Tab` > `Add` to open a new configuration form, then fill up the necessary fields as follows:

| Edit Advanced Outbound NAT Entry | Value | Value
| :---  | :--- | :--- |
| Disabled | `[]` Disable this rule
| Do not NAT | `[]` Enabling this option will .....
| Interface | `VPNGATELOCAL`
|Address Family | `IPv4+IPv6`
| Protocol | `any`
| Source | `Network` | `192.168.40.0/24`
| Destination | `Any` | Leave blank
| **Translation**
| Address | `Interface Address`
| Port or Range | Leave blank
| **Misc**
| No XMLRPC Sync | `[]`
| Description | `VLAN40 to vpngate-local`

And click `Save`.

![alt text](https://raw.githubusercontent.com/ahuacate/pfsense-setup/master/images/pfsense_nat_02.png)

Now your first two mappings for the new gateways show look like this:

| |Interface|Source|Source Port|Destination|Destination Port|NAT Address|NAT Port|Static Port|Description|
|:--- | :---  | :--- | :--- | :---  | :--- | :--- | :---  | :--- | :--- |
|[]|VPNGATEWORLD|192.168.30.0/24|*|*|*|VPNGATEWORLD address|*|:heavy_check_mark:|VLAN30 to vpngate-world
|[]|VPNGATELOCAL|192.168.40.0/24|*|*|*|VPNGATELOCAL address|*|:heavy_check_mark:|VLAN40 to vpngate-local

## 8.00 Adding Firewall Rules
This is simple because we are going to send all the traffic in a subnet(s) (VLAN30 > vpngate-world / VLAN40 > vpngate-local) through the openVPN tunnel. And remember Firewall rules work top down so best set `Allow Rules` above `Block Rules` below.

### 8.01 Allow Rule for OPT1 - vpngate-world
Go to  `Firewall` > `Rules` > `OPT1 tab` and `Add` a new rule:

| Edit Firewall Rule / OPT1 | Value | Notes|
| :---  | :--- | :--- |
| Action | `Pass`
| Disabled | `[]` disable this rule
| Interface | `OPT1`
| Addresss Family | `IPv4+IPv6`
| Protocol | `Any`
| **Extra Options**
| Log | `[]` Log packets that are handled by this rule
| Description | `VLAN30 Traffic to vpngate-world`
| Advanced Options | Click `Display Advanced` | *This is a important step. You only want to edit one value in `Advanced`!!!!*
| **Advanced Options**
| Gateway | `VPNGATEWORLD_VPNV$-x.x.x.x-Interface VPNGATEWORLD_VPNV4 Gateway` | *MUST Change to this gateway!*

Click `Save`.

### 8.02 Allow Rule for OPT2 - vpngate-local
Go to  `Firewall` > `Rules` > `OPT2 tab` and `Add` a new rule:

| Edit Firewall Rule / OPT2 | Value | Notes|
| :---  | :--- | :--- |
| Action | `Pass`
| Disabled | `[]` disable this rule
| Interface | `OPT2`
| Addresss Family | `IPv4+IPv6`
| Protocol | `Any`
| **Extra Options**
| Log | `[]` Log packets that are handled by this rule
| Description | `VLAN40 Traffic to vpngate-local`
| Advanced Options | Click `Display Advanced` | *This is a important step. You only want to edit one value in `Advanced`!!!!*
| **Advanced Options**
| Gateway | `VPNGATELOCAL_VPNV4-x.x.x.x-Interface VPNGATELOCAL_VPNV4 Gateway` | *MUST Change to this gateway!*

Click `Save` and `Apply`.

The above two rules will send all the traffic on that interface into the VPN tunnel, you must ensure that the ‘gateway’ option is set to your VPN gateway and that this rule is above any other rule that allows hosts to go out to the internet. pfSense needs to be able to catch this rule before any others.

### 8.03 Redirect DNS on OPT1 - vpngate-world
When your computer needs to know an IP Address of a host it will use a DNS server and by default it will use your internet service providers or the DNS resolver built into pfSense. This is bad when using a VPN because performing a DNS lookup can reveal your origin IP Address.

To fix this problem we’re going to create a basic port forward rule which will take traffic destined for UDP port 53 (the DNS server port) and forward it to a different DNS server that is operated by your VPN provider. In this example we will redirect DNS 53 to ExpressVPN DNS server IP address.

Go to  `Firewall` > `Rules` > `OPT1 tab` and `Add (arrow down)` a new rule:

| Firewall Rule / OPT1 | Value | Notes
| :---  | :--- | :---
| **Edit Firewall Rule **
| Action | `Pass`
| Disabled | `[]` disable this rule
| Interface | `OPT1`
| Addresss Family | `IPv4`
| Protocol | `TCP/UDP`
| **Source**
| Source 
| | `[]` Invert match. 
| | `OPT1 net` 
| | Source Address - leave blank
| **Destination**
| Destination 
| | `[]` Invert match.
| | `Single host or alias` 
| | `85.203.37.1` | *Change to your VPN providers DNS*
| Destination Port Range
| | From `DNS (53)`
| | Custom `Leave Blank`
| | To `DNS (53)`
| | Custom `Leave Blank`
| **Extra Options**
| Log | `[]` Log packets that are handled by this rule
| Description | `VLAN40 Traffic to vpngate-world`
| Advanced Options | Leave Default

Click `Save` and `Apply`.

![alt text](https://raw.githubusercontent.com/ahuacate/pfsense-setup/master/images/pfsense_rules_01.png)

### 8.04 Redirect DNS on OPT2 - vpngate-local
When your computer needs to know an IP Address of a host it will use a DNS server and by default it will use your internet service providers or the DNS resolver built into pfSense. This is bad when using a VPN because performing a DNS lookup can reveal your origin IP Address.

To fix this problem we’re going to create a basic port forward rule which will take traffic destined for UDP port 53 (the DNS server port) and forward it to a different DNS server that is operated by your VPN provider. In this example we will redirect DNS 53 to ExpressVPN DNS server IP address.

Go to  `Firewall` > `Rules` > `OPT2 tab` and `Add (arrow down)` a new rule:

| Firewall Rule / OPT2 | Value | Notes
| :---  | :--- | :---
| **Edit Firewall Rule **
| Action | `Pass`
| Disabled | `[]` disable this rule
| Interface | `OPT2`
| Addresss Family | `IPv4`
| Protocol | `TCP/UDP`
| **Source**
| Source 
| | `[]` Invert match. 
| | `OPT2 net` 
| | Source Address - leave blank
| **Destination**
| Destination 
| | `[]` Invert match.
| | `Single host or alias` 
| | `85.203.37.1` | *Change to your VPN providers DNS*
| Destination Port Range
| | From `DNS (53)`
| | Custom `Leave Blank`
| | To `DNS (53)`
| | Custom `Leave Blank`
| **Extra Options**
| Log | `[]` Log packets that are handled by this rule
| Description | `VLAN40 Traffic to vpngate-local`
| Advanced Options | Leave Default

Click `Save` and `Apply`.

![alt text](https://raw.githubusercontent.com/ahuacate/pfsense-setup/master/images/pfsense_rules_02.png)

### 8.05 Allow OPT2 access to Chromecast and TV devices - vpngate-local
By default VLAN40 (vpngate-local) has no access to any other other VLAN. For convenience we allow certain VLAN40 ports access to our Chromecast Alias IP address.

Go to  `Firewall` > `Rules` > `OPT2 tab` and `Add (arrow up)` a new rule:

| Firewall Rule / OPT2 | Value | Notes
| :---  | :--- | :---
| **Edit Firewall Rule **
| Action | `Pass`
| Disabled | `[]` disable this rule
| Interface | `OPT2`
| Addresss Family | `IPv4`
| Protocol | `TCP`
| **Source**
| Source 
| | `[]` Invert match. 
| | `OPT2 net` 
| | Source Address - leave blank
| **Destination**
| Destination 
| | `[]` Invert match.
| | `Single host or alias` 
| | `Chromecast_IP_Addresses` | *Here we use the Firewall alias `Chromecast_IP_Addresses` we created before*
| Destination Port Range
| | From `Other`
| | Custom `8008`
| | To `Other`
| | Custom `8009`
| **Extra Options**
| Log | `[]` Log packets that are handled by this rule
| Description | `Allow OPT2 (VPNGate-local) Port 8008-8009 to Chromecast`
| Advanced Options | Leave Default

Click `Save`.

Now we need to create another rule. Go to  `Firewall` > `Rules` > `OPT2 tab` and `Add (arrow up)` a new rule:

| Firewall Rule / OPT2 | Value | Notes
| :---  | :--- | :---
| **Edit Firewall Rule **
| Action | `Pass`
| Disabled | `[]` disable this rule
| Interface | `OPT2`
| Addresss Family | `IPv4`
| Protocol | `UDP`
| **Source**
| Source 
| | `[]` Invert match. 
| | `OPT2 net` 
| | Source Address - leave blank
| **Destination**
| Destination 
| | `[]` Invert match.
| | `Single host or alias` 
| | `224.0.0.251` | 
| Destination Port Range
| | From `Other`
| | Custom `1900`
| | To `Other`
| | Custom `1900`
| **Extra Options**
| Log | `[]` Log packets that are handled by this rule
| Description | `Allow OPT2 (VPNGate-local) Port 1900 to Chromecast`
| Advanced Options | Leave Default

Click `Save`.

Now we need to create another rule. Go to  `Firewall` > `Rules` > `OPT2 tab` and `Add (arrow up)` a new rule:

| Firewall Rule / OPT2 | Value | Notes
| :---  | :--- | :---
| **Edit Firewall Rule **
| Action | `Pass`
| Disabled | `[]` disable this rule
| Interface | `OPT2`
| Addresss Family | `IPv4`
| Protocol | `UDP`
| **Source**
| Source 
| | `[]` Invert match. 
| | `OPT2 net` 
| | Source Address - leave blank
| **Destination**
| Destination 
| | `[]` Invert match.
| | `Single host or alias` 
| | `224.0.0.251` | 
| Destination Port Range
| | From `Other`
| | Custom `5353`
| | To `Other`
| | Custom `5353`
| **Extra Options**
| Log | `[]` Log packets that are handled by this rule
| Description | `Allow OPT2 (VPNGate-local) Port 5353 to Chromecast`
| Advanced Options | Leave Default

Click `Save` and `Apply`.

Your `Firewall` > `Rules` > `OPT2 tab` should now look like:

![alt text](https://raw.githubusercontent.com/ahuacate/pfsense-setup/master/images/pfsense_rules_00.png)

## 8.00 Setup pfSense DNS
Here will setup two DNS services.

### 8.01 Set Up DNS Resolver
To configure the pfSense DNS resolver navigate to `Services` > `DNS Resolver` and on the tab `General Settings` fill up the necessary fields as follows:

| DNS Resolver Settings | Value | Notes
| :---  | :--- | :--- 
| **General Settings Tab**
| **General DNS Resolver Options**
| Enable | `☑ Enable DNS resolver` | 
| Listen Port | Leave Default | 
| Enable SSL/TLS Service | `[]` | 
| SSL/TLS Certificate | Leave Default | 
| SSL/TLS Listen Port | Leave Default | *Should be 853 by default*
| Network Interfaces | `LAN` | *Select ONLY LAN, OPT1, OPT2 and Localhost - Use the Ctrl key to toggle selection*
||`OPT1`
||`OPT2`
||`Localhost`
| Outgoing Network Interfaces | `VPNGATEWORLD` |*Select ONLY VPNGATEWORLD and VPNGATELOCAL - Use the Ctrl key to toggle selection*
|| `VPNGATELOCAL`
| System Domain Local Zone Type | `Static` | 
| DNSSEC | `☑ Enable DNSSEC Support` | 
| DNS Query Forwarding | `[]` Enable Forwarding Mode | *Uncheck*
|| `[]` Use SSL/TLS for outgoing DNS Queries to Forwarding Servers | *Uncheck*
| DHCP Registration | `[]` Register DHCP leases in the DNS Resolver | *Uncheck*
| Static DHCP | `[]` Register DHCP static mappings in the DNS Resolver | *Uncheck*
| OpenVPN Clients | `[x]` Register connected OpenVPN clients in the DNS Resolver | *Check*
| Display Custom Options | Click `Display Custom Options` | 
| Custom options  | `server:include: /var/unbound/pfb_dnsbl.*conf`
| ** Advanced Settings Tab**
| **Advanced Pricvacy Options**
| Hide Identity | `[x]` id.server and hostname.bind queries are refused
| Hide Version | `[x]` version.server and version.bind queries are refused
| Query Name Minimization | `[]` Send minimum amount of QNAME/QTYPE information to upstream servers to enhance privacy
| Strict Query Name Minimization | `[]` Do not fall-back to sending full QNAME to potentially broken DNS servers
| **Advanced Resolver Options**
| Prefetch Support | `[]` Message cache elements are prefetched before they expire to help keep the cache up to date
| Prefetch DNS Key Support | `[x]` DNSKEYs are fetched earlier in the validation process when a Delegation signer is encountered
| Harden DNSSEC Data | `[x]` DNSSEC data is required for trust-anchored zones.

And click `Save`.

### 8.02 Set Up General DNS

Navigate to `System` > `General Settings` and fill up the necessary fields as follows:

| General Setup | Value | Value | Notes
| :---  | :--- | :--- | :---
| **System**
| Hostname | `pfSense`
| Domain | `localdomain`
| **DNS Server Settings**
| DNS Servers | `1.1.1.1` | DNS Hostname - Leave Default | `none` | *Must choose none*
| DNS Servers | `1.0.0.1` | DNS Hostname - Leave Default | `none` | *Must choose none*
| DNS Server Override | `[]` Allow DNS server list to be overridden by DHCP/PPP on WAN
| Disable DNS Forwarde | `[x]` Do not use the DNS Forwarder/DNS Resolver as a DNS server for the firewall
| **Localisation**
| Timezone | Select your region
| Timeservers | 0.pfsense.pool.ntp.org
| Language | Select your language

And click `Save`. Your pfSense appliance is now configured for DNS servers.

## 9.00 Setup NTP
This is easy. Navigate to `Services` > `NTP` >`Settings` and fill up the necessary fields as follows:

![alt text](https://raw.githubusercontent.com/ahuacate/pfsense-setup/master/images/pfsense_ntp_01.png)

## 10.00 Setup Gateway Monitoring

## 10.00 Finish Up
After all your rules are in place head over to `Diagnostics` > `States` > `Reset States Tab` > and tick `Reset the firewall state table` click `Reset`. After doing any firewall changes that involve a gateway change its best doing a state reset before checking if anything has worked (odds are it will not work if you dont). PfSense WebGUI may hang for period but dont despair because it will return in a few seconds for routing to come back and up to a minute, don’t panic.

And finally navigate to `Diagnostics` > `Reboot` and reboot your pfSense machine.

Once you’re done head over to any client PC on the network or mobile on the WiFi SSID on either `vpngate-world` VLAN30 or `vpngate-local` VLAN40 networks and use IP checker to if its all working https://wtfismyip.com with NO DNS Leaks https://www.dnsleaktest.com/ (use the extended option on dnsleaktest).

Success! (hopefully)

---

## 00.00 Patches and Fixes

## 8.0 Install & Setup pfBlockerNG on pfSense
pfBlockerNG can add other security enhancements such as blocking known bad IP addresses with blocklists. For example, getting rid of adverts and pop-ups from websites. If you don’t already have a blocklist functionality in place on your pfSense (such as PiHole), I would strongly suggest adding pfBlockerNG Devel to your new OpenVPN Gateways (VPNGGATEWORLD and VPNGATELOCAL).

### 8.1 pfBlockerNG Installation
In the pfSense WebGUI go to `System` > `Package Manager` > `Available Packages` and type ‘pfblocker’ into the search criteria and then click `Search`.

Make sure you click `+ Install` on the version with ‘-devel’ (i.e pfBlockerNG-devel) at the end of it, and then `Confirm` on the next page. Installation may take a short while as it downloads and updates certain packages.  

At this point, you have already installed the package. Next, you will need to enable it from pfSense WebGUI `Firewall` > `pfBlockerNG` and the option to exit out of the wizard. A configuration page should appear, Click on the `General Tab`, and fill out the necessary fields as follows:

| General Settings | Value | Value | Value | Value | Notes
| :---  | :--- | :--- | :--- | :--- | :---
| pfBlockerNG | `☑` Enable | 
| Keep Settings | `☑` Enable |
| CRON Settings | Every hour | 00 | 0 | 0 | *Generally Leave Default settings*

Then Click `Save` at the bottom of the page.

### 8.2 Configure IP
Go to pfSense WebGUI `Firewall` > `pfBlockerNG` > `IP Tab` and fill out the necessary fields as follows. Whats NOT shown in the below table leave as default.

| IP Configuration | Value | Other Values | Notes
| :---  | :--- | :--- | :---
| De-Duplication | `☑` Enable | |*Check*
| CIDR Aggregation | `☑` Enable ||*Check*
| Suppression | `☑` Enable ||*Check*
| Global Logging | [] ||*Uncheck*
| Placeholder IP Address | 127.1.7.7||*Leave Default*
| MaxMind Localized Language | English||*Leave Default*
| MaxMind Updates | [] Check to disable MaxMind updates ||*Uncheck*
| Global Logging | [] ||*Uncheck*
| **IP Interface/Rules Configuration**
| Inbound Firewall Rules | `VPNGATEWORLD` || *Select ONLY VPNGATEWORLD and VPNGATELOCAL*
|| `VPNGATELOCAL`
| Outbound Firewall Rules | `VPNGATEWORLD` || *Select ONLY VPNGATEWORLD and VPNGATELOCAL*
|| `VPNGATELOCAL`
| Floating Rules | ☑ Enabled || *Check*
| Firewall 'Auto' Rule Order | pfB_Pass/Match/Block/Reject\All other Rules\(Default Format) || *Leave Default*
| Firewall 'Auto' Rule Suffix | `auto rule`
| Kill States | `☑` Enable

### 8.3 Configure DNSBL
Because we have multiple internal interfaces, we are using a Qotom Mini PC Q500G6-S05 with 6x Gigabit NICs, you would want to protect them with DNSBL, so you will need to pay attention to the ‘Permit Firewall Rules’ section. First, place a checkmark in the ‘Enable’ box of `Permit Firewall Rules`. Then, select the various interfaces (to the right, in a box) by holding down the ‘Ctrl’ key and left-click selecting the interfaces you choose to protect with pfBlockerNG. Note, don’t forget to Click the `Save DNSBL settings` at the bottom of the page.

Also if your pfSense OS has plenty of memory enable, 3Gb or more, use TLD. Normally, DNSBL (and other DNS blackhole software) block the domains specified in the feeds and that’s that. What TLD does differently is it will block the domain specified in addition to all of a domain’s subdomains. As a result, a instrusive domain can’t circumvent the blacklist by creating a random subdomain name such as abcd1234.zuckermine.com (if zuckermine.com was in a DNSBL feed). If you have the RAM enable it.

Next go to pfSense WebGUI `Firewall` > `pfBlockerNG` > `DNSBL Tab` and fill out the necessary fields as follows. Whats NOT shown in the below table leave as default. 

| DNSBL | Value | Other Values | Notes
| :---  | :--- | :--- | :---
| DNSBL | `☑` Enable | 
| TLD | `☑` Enable | | *Note: You need at least 3Gb of RAM for this feature*
| **DNSBL Webserver Configuration**
| Virtual IP Address | 10.10.10.1 || *Leave Default*
| VIP Address Type | IP Alias | Leave Blank (Enter Carp Password) | *Leave Default*
| Port | 8081 || *Leave Default*
| SSL Port | 8443 || *Leave Default*
| Webserver Interface | LAN || *Leave Default*
| **DNSBL Configuration**
| Permit Firewall Rules | `☑` Enable |`OPT1` | *Select ONLY OPT1 and OPT2 - Use the Ctrl key to toggle selection*
||| `OPT2`
| Blocked Webpage | dnsbl_default.php || *Leave Default*
| Resolver Live Sync | `[]` Enable || *Uncheck*
| **DNSBL IPs**
| List Action | `Deny Outbound`
| Enable Logging | `Enable`

Now Click `Save DNSBL settings` at the bottom of the page.

### 8.4 Configure DNSBL feeds
Using the pfSense WebGUI  `Firewall` > `pfBlockerNG` > `Feeds Tab` (not DNSBL Feeds) at the top. Here you will see all of the pre-configured feeds for the IPv4, IPv6, and DNSBL categories.

Scroll down to the `DNSBL Category` header then to the Alias/Group labeled `ADs`. Click the blue colour **`+`** next to the `ADs` header (column should be all ADs) to add all the feeds related to ADs category. Note, if you instead clicked the `+` to the far right of each line, you will instead only add that individual feed - this is not what we want.

| Category | Alias/Group | Feed/Website | Header/URL
| :---  | :--- | :--- | :---
| **DNSBL Category**
| DNSBL `I` **`+`** | `ADs` | `Adaway` | `Adaway`

If you clicked the **`+`** next to the ADs category, you are taken to a `DNSBL feeds` page with all of the feeds under that category pre-populated. All of the feeds in the list will initially be in the `OFF` state. You can go through and enable each one individually or you can click `Enable All` at the bottom of the list - then all will switch/change to `ON` state. Then change the Action field to `Unbound`.

Next, make sure you switch the `Action` from Disabled to `Unbound`.

| Value | Value | Value | Value
| :---  | :--- | :--- | :---
| **DNSBL Source Definitions**
| `Auto` | `ON` | `https://adaway.org/hosts.txt` | `Adaway`
| **Settings**
| Action | `Unbound`

Now Click the `Save DNSBL Settings` at the bottom of the page and you should receive a message at the top along the lines of `Saved [ Type:DNSBL, Name:ADs ] configuration`.

To check all went okay go to the `Firewall` > `pfBlockerNG` > `DNSBL` > `DNSBL Feeds` tab and you will see a DNSBL feeds summary. Your feeds summary should look similar to the one below:

| Name | Description | Action | Frequency | Logging
| :---  | :--- | :--- | :--- | :---
| **DNSBL Feeds Summary**
| ADs | ADs - Collection | Unbound | Once a day | Enabled

Now lets add some more. Go back to `Firewall` > `pfBlockerNG` > `Feeds` tab up top and then scroll down to `DNSBL category` section again. We’re going to add another category (after making some changes), but let’s explain everything you see here. Looking at the `DNSBL Catergory` you’ll see the `ADs` category checkmark **`+`** is replaced with a :heavy_check_mark: means this category already exists and is active in the DNSBL ADs category. This distinction is important to recognize because when you add the next category we do not need to enable every feed for a particular category.

Also worth mentioning before we add the `Malicious` category. Some feeds have selectable options such as feed category `Internet Storm Center`. I recommend switching the feed from `ISC_SDH` (high) to `ISC_SDL` (low) as the high feed has under 20 entries and the low feed includes the high feed.

After making the switch to `ISC_SDL`, click the blue colour **`+`** next to the `Malicious` header (column should be all Malicious) to add all the feeds related to that category.
If you clicked the **`+`** next to the Makicious category, you are taken to a `DNSBL feeds` page with all of the feeds under that category pre-populated. As when we added the ADs list, go ahead and click `Enable All` at the bottom of the list - all will switchchange to `ON` state. Then change the Action field to `Unbound`. **Don’t hit save just yet!**

**Important:** Now look for any `Header/label` called **`Pulsedive`** and/or **`Malekal`** and delete them (they were not there in my pfBlockerNG version). You don't want these as they are subscription (paid) services. On deletion they will disappear.

Now Click the `Save DNSBL Settings` at the bottom of the page and you should receive a message at the top along the lines of `Saved [ Type:DNSBL, Name:ADs ] configuration`.

To check all went okay go to the `Firewall` > `pfBlockerNG` > `DNSBL` > `DNSBL Feeds` tab and you will see a DNSBL feeds summary. Your feeds summary should look similar to the one below:

| Name | Description | Action | Frequency | Logging
| :---  | :--- | :--- | :--- | :---
| **DNSBL Feeds Summary**
| ADs | ADs - Collection | Unbound | Once a day | Enabled
| Malicious | Malicious - Collection | Unbound | Once a day | Enabled

Now lets add some more. Scroll down to the `DNSBL Category` header then to the Alias/Group labeled `Easylist`. Click the blue colour **`+`** next to the `Easylist` header (column should be all Easylist) to add all the feeds related to that category. You are taken to a `DNSBL feeds` page with all of the feeds under that category pre-populated. 

**Important:** Now look for any `Header/label` called `EasyPrivacy` and delete it. On deletion the line will disappear.

All of the feeds in the list will initially be in the `OFF` state. You can go through and enable each one individually or you can click `Enable All` at the bottom of the list - all will switch/change to `ON` state. Then change the Action field to `Unbound` and the Update Frequency to `Every 4 hours`.

Now Click the `Save DNSBL Settings` at the bottom of the page and you should receive a message at the top along the lines of `Saved [ Type:DNSBL, Name:ADs ] configuration`.

Now repeat the procedure for:
*  BBcan177 - From the creater of pfBlockerBG
*  hpHosts (all of them) - From Malwarebytes
*  BBC (BBC_DGA_Agr) – From Bambenek Consulting
*  Cryptojackers (all of them) – This blocks cryptojacking software and in-browser miners, but it also blocks various coin exchanges.

After adding all of the above go to the `Firewall` > `pfBlockerNG` > `DNSBL` > `DNSBL Feeds` tab and you will see a DNSBL feeds summary. Your feeds summary should look similar to the one below:

| Name | Description | Action | Frequency | Logging
| :---  | :--- | :--- | :--- | :---
| **DNSBL Feeds Summary**
| ADs | ADs - Collection | Unbound | Once a day | Enabled
| Malicious | Malicious - Collection | Unbound | Once a day | Enabled
| hpHosts | Malwarebytes - Collection | Unbound | Once a day | Enabled
| BBcan177 | BBcan177 - Collection | Unbound | Once a day | Enabled
| BBC | BBC-DGA type - Collection | Unbound | Once a day | Enabled
| Cryptojackers | Cryptojackers - Collection | Unbound | Once a day | Enabled
| Easylist | EasyList Feeds | Unbound | Every 4 hours | Enabled

### 8.5 Force DNSBL Feed Updates
You need to force a update to to `Reload` DNSBL new or changed settings. You must do this to check if your pfBlockerNG is working.

Next go to pfSense WebGUI `Firewall` > `pfBlockerNG` > `Update Tab` and fill out the necessary fields as follows. Whats NOT shown in the below table leave as default. 

| Update Settings | Value | Vale | Value | Notes
| :---  | :--- | :--- | :--- | :---
| Links 
| Select Force option | [] Update | [] Cron | `☑` Reload | *Select Reload*
| Select Reload option | `☑` All | [] IP | [] DNSBL | *Select All*

Now Click the `RUN` below the options and you should see the Logs being created on the page. It may take a while. Be patient.

### 8.6 Check if pfBlockerNG is working
First connect a device (i.e mobile, tablet etc) to either *.vpngate-local or *vpngate-world network. Go and browse a few websites like a news website. Then go to pfSense WebGUI `Firewall` > `pfBlockerNG` > `Reports` > `Alerts Tab` and you should see the DNSBL entry being populated with intercepted data. 

| Date | IF | Source | Domain/Referer/URI/Agent | Feed
| :---  | :--- | :--- | :--- | :---
| **DNSBL Section**
 Jul 26 10:49:13 [65]|OPT2|192.168.40.151|graph.instagram.com [ DNSBL ]S|Yoyo
 ||| Galaxy-Note8|DNSBL-HTTP|DNSBL_ADs
|Jul 26 11:06:05|OPT2|192.168.40.151|connect.facebook.net [ TLD ]|AntiSocial_BD
||||DNSBL-HTTPS | |DNSBL_Malicious

If you see nothing in the DNSBL section then pfBlockerNG is NOT working. Check your configurations to resolve. Remember after any edits or changes always perform a pfBlockerNG Update by following the procedures in **8.5 Force DNSBL Feed Updates**.

If I am left scratching my heading wondering what I've done wrong I find deleting and recreating the pfSense firewall floating rules often fixes things. My procedure is as follows:
*  Step 1: Go to pfSense WebGUI `Firewall` > `pfBlockerNG` > `General Tab` and disable pfBlockerNG. Click `Save` at the bottom of the page.
*  Step 2: Next go to pfSense WebGUI `Firewall` > `pfBlockerNG` > `DNSBL Tab` and disable DNSBL. Click `Save` at the bottom of the page.
*  Step 3: Next go to pfSense WebGUI `Firewall` > `Rules` > `Floating Tab` and delete all 3 rules and click `Save`.
*  Step 4: Next go to pfSense WebGUI `Diagnostics` > `States` > `Reset States` select `Reset the firewall state table` and click `Reset`.
*  Step 5: Re-enable `pfBlockerNG` and `DNSBL` shown in Step 1 and 2.
*  Step 6: Now perform a pfBlockerNG Update by following the procedures in **8.5 Force DNSBL Feed Updates**.

## 9.0 Create a pfSense Backup
If all is working its best to make a backup of your pfsense configuration. Also if you experiment around a lot, it’s an easy way to restore back to a working configuration. Also, do a backup each and every time before upgrading to a newer version of your firewall or pfSense OS. So in the event you have to rebuild pfSense you can skip Steps 7.0 onwards by using the backup restore feature which will save you a lot of time.

On your pfSense WebGUI navigate to `Diagnostics` > `Backup & Restore` then fill up the necessary fields as follows:

| Backup Configuration | Value | Notes
| :---  | :--- | :--- |
| Backup area | `All` | *Select All*
| Skip Packages | `[]` Do not backup package information | *Uncheck the box*
| Skip RRD data | `☑` Do not backup RRD data (NOTE: RRD Data can consume 4+ megabytes of config.xml space!)| *Check the box. RRD Data are your Graphs. Like the traffic Graph for example. I do not back them up because I do not need them*
| Encyption | `[]` Encrypt this configuration file. | *If you check this box a password value box will appear. Dont forget your password otherwise you are truly stuffed if you need to perform a restore*
| Password (optional box) | `xxxxxxxx` | *See above*

And then click the `Download configuration as XML` and `Save` the backup XML file to your NAS or a secure location. Note: If you are using the WebGUI on a Win10 PC the XML backup file will be saved in your users `Downloads` folder where you can then copy/move the file to a safer location. You should have a backup folder share on your NAS so why not store the XML file there `backup/pfsense/config-pfSense.localdomain-2019xxxxxxxxxx.xml`

## 10.0 Create a Cluster
At this stage you should have 3x fully built and ready Proxmox nodes on the same network - Typhoon-01, Typhoon-02 and Typhoon-03. You can need create a 3x node cluster.

### 10.1 Create the Cluster
Now using the pfSense web interface on node-01, Typhoon-01, go to `Datacenter` > `Cluster` > `Create Cluster` and fill out the fields as follows:

| Create Cluster | Value | Notes
| :---  | :--- | :--- |
| Cluster Name | `typhoon-cluster` |
| Ring 0 Address | Leave Blank |

And Click `Create`.

### 10.2 Join the other Nodes to the New Cluster
The first step in joining other nodes to your cluster, `typhoon-cluster`, is to copy typhoon-01 cluster manager fingerprint/join information into your clipboard.

**Step One:**

Now using the pfSense web interface on node-01, Typhoon-01, go to `Datacenter` > `Cluster` > `Join Information` and a new window will appear showing `Cluster Join Information` with the option to `Copy Information` into your clipboard. Click `Copy Information`.

**Step Two:**

Now using the pfSense web interface on the OTHER Nodes, Typhoon-02/03/04 etc, go to `Datacenter` > `Cluster` > `Join Cluster` and a new window will appear showing `Cluster Join` with the option to paste the `Cluster Join Information` into a `Infoprmation` field. Paste the information, enter your root password into the `Password` field and the other fields will automatically be filled.

And  Click `Join`. Repeat for on all nodes.

The result should be any configuration can now be done via a single Proxmox web interface. Just point your browser to the IP address given during installation (https://192.168.1.101:8006) and all 3x nodes should be shown below `Datacenter (typhoon-cluster)`. Or type `pvecm status` into any node `typhoon-01` > `>_Shell`:

```
pvecm status

# Results ...
Quorum information
------------------
Date:             Mon Jul 22 13:44:10 2019
Quorum provider:  corosync_votequorum
Nodes:            3
Node ID:          0x00000001
Ring ID:          1/348
Quorate:          Yes

Votequorum information
----------------------
Expected votes:   3
Highest expected: 3
Total votes:      3
Quorum:           2  
Flags:            Quorate 

Membership information
----------------------
    Nodeid      Votes Name
0x00000001          1 192.168.1.101 (local)
0x00000002          1 192.168.1.102
0x00000003          1 192.168.1.103
```

### 10.3 How to delete a existing cluster on a node
I made an error when creating the cluster name and it was headache to delete the cluster. But if you paste the following into a CLI terminal your cluster settings should be reset to default.

```
systemctl stop pve-cluster &&
pmxcfs -l &&
rm -f /etc/pve/cluster.conf /etc/pve/corosync.conf &&
rm -f /etc/cluster/cluster.conf /etc/corosync/corosync.conf &&
systemctl stop pve-cluster &&
rm /var/lib/pve-cluster/.pmxcfs.lockfile &&
rm -f /etc/corosync/authkey &&
systemctl start pve-cluster &&
systemctl restart pvedaemon &&
systemctl restart pveproxy &&
systemctl restart pvestatd &&
reboot
```

### 11.0 pfSense – disable firewall with pfctl -d
If for whatever reason you have lost access to the pfSense web management console then go to the Proxmox web interface `typhoon-01` > `251 (pfsense)` > `>_ Console` and `Enter an option` numerical `8` to open a shell.

Then type and execute `pfctl -d` where the -d will temporally disable the firewall (you should see the confirmation in the shell `pf disabled`, where pf is the packet filter = FIREWALL)

Now you can log into the WAN side IP address (192.168.2.1) and govern the pfsense again to fix the problem causing pfSense web management console to sease working on 192.168.1.253.
