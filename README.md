# REPORT

# 1. Installation of Debian Without a Graphical Interface

I started by creating 2 virtual machines (VMs), each with 8 GB of RAM and 2 CPU cores. Then, I launched the Debian installation in classic mode, choosing not to install GNOME or MATE.

# 2. System Updates

To update the systems, I first allowed my user "samuel" to be in the sudoers group using the following commands:

    su -
    
    usermod -aG sudo Samuel
    
    exit

Then, I executed `sudo apt update` and `sudo apt upgrade` to perform the updates.

# 3. DHCP Server Configuration

First, I updated the system with:

    sudo apt update && apt upgrade -y

Next, I installed the `isc-dhcp-server` package with the command:

    sudo apt install isc-dhcp-server -y

I edited the DHCP configuration file with:

    sudo nano /etc/dhcp/dhcpd.conf

And added the following lines:

    subnet 172.16.0.0 netmask 255.255.0.0 {
    
    range 172.16.1.2 172.16.2.253;
    
    option routers 172.16.0.1;
    
    option domain-name-servers 8.8.8.8, 8.8.4.4;
    
    }

Then, I specified the DHCP network interface with:

    nano /etc/default/isc-dhcp-server

And in the line `INTERFACESv4=""`, I added `ens33`.

After that, I configured the network interface with:

    nano /etc/network/interfaces

And added:

    auto ens33
    
    iface ens33 inet static
    
    address 172.16.0.2
    
    netmask 255.255.0.0
    
    gateway 192.168.127.55 # Gateway I found by running `ip route` on Debian

Next, I applied the network changes by restarting the network card:

    ifdown ens33 && ifup ens33

Finally, I started the DHCP service with:

    systemctl start isc-dhcp-server
    
    systemctl status isc-dhcp-server

# 4. Installation of FTP and SSH Servers

## SSH:

First, I launched my VM2 and installed Debian, including the SSH server by default. Then, I performed an update with:

    apt update && apt upgrade -y

Next, I executed the following commands as the superuser (`su -`):

    systemctl start ssh
    
    systemctl enable ssh

After that, I went to PowerShell on my Windows host to access my VM via SSH. I executed:

    ssh samuel@172.16.0.102

After entering my password, the connection worked. From my terminal controlling the VM, I began preparing the FTP connections.

## FTP:

I installed the FTP server with:

    apt install proftpd -y

Then, I started it with:

    systemctl start proftpd
    
    systemctl enable proftpd

I followed the IT-Connect tutorial to configure the FTP server. I started by creating a configuration file for FTP:

    nano /etc/proftpd/conf.d/ftp-perso.conf

And filled it with the following data:

    # Server name (same as defined in /etc/hosts)
    
    ServerName "debianHTPSSH"
    
    # Login message
    
    DisplayLogin "Welcome to Samuel's FTP!"
    
    # Disable IPv6
    
    UseIPv6 off
    
    # Each user only accesses their home directory (for members of the ftp2100 group)
    
    DefaultRoot ~ ftp2100
    
    # Port (default = 21)
    
    Port 21
    
    # Deny root login
    
    RootLogin off
    
    # Maximum number of FTP clients
    
    MaxClients 5
    
    # Allow connections only for members of the "ftp2100" group using the DenyGroup directive.
    
    # By specifying "!", everything is denied except the "ftp2100" group
    
    <Limit LOGIN>
    
    DenyGroup !ftp2100
    
    </Limit>

Then, I saved and restarted the FTP service with:

    systemctl reload proftpd

Next, I created a new user who would have exclusive access to the FTP. I started by creating the group they would belong to:

    addgroup ftp2100

Then, I created the user:

    adduser laplateforme

I entered the password `Marseille13!` as specified in the subject.

After that, I added the user to the group:

    adduser laplateforme ftp2100

This allowed the user `laplateforme` to access the FTP, as only members of the `ftp2100` group can access it.

To test the connection, I proceeded in two ways:

### 1. From another CMD window:

I executed:

    ftp 172.16.0.102

Then, I entered the credentials:

- Username: laplateforme

- Password: Marseille13!

This allowed me to access the FTP, but I couldn't find a functional command to modify files.

### 2. Via FileZilla:

I downloaded FileZilla on my host PC. I connected using the same IP address and credentials. From FileZilla, I could add and delete files, which then appeared in the VM.

#### SFTP:

To secure my SSH access using SFTP, I started by modifying the SSH configuration file with the command:

    nano /etc/ssh/sshd_config

I verified that the following line was present:

    Subsystem sftp /usr/lib/openssh/sftp-server

Then, I restricted certain users to SFTP only by adding the following at the end of the file:

    Match Group sftpusers
    
    ChrootDirectory /home/%u
    
    ForceCommand internal-sftp
    
    AllowTcpForwarding no
    
    X11Forwarding no

This limits users in the `sftpusers` group to their home directory and prevents them from accessing a shell.

I finished by restarting the SSH service to apply the changes:

    sudo systemctl restart sshd

I wanted to separate FTP and SFTP users, so I created the SFTP group with:

    sudo groupadd sftpusers

Then, I created a new user for SFTP and added them to the `sftpusers` group with the following commands:

    sudo adduser samuelsftp

I entered the password `root`, then executed:

    sudo adduser samuelsftp sftpusers

I disabled the FTP server to avoid errors:

    sudo systemctl stop proftpd

# 5. DNS Server Installation

For the DNS server installation, I positioned myself on my first VM, which already has the DHCP server. I installed Bind9 as follows:

    sudo apt install bind9 bind9-utils bind9-doc

Next, I configured the DNS server with the following command:

    sudo nano /etc/bind/named.conf.options

I replaced the existing content with:

    options {
    directory "/var/cache/bind";
    recursion yes;  # Allows recursion for clients
    allow-query { 172.16.0.0/24; };  # Allows queries from the 172.16.0.0/24 network
    forwarders {
    8.8.8.8;  # Uses Google DNS as a forwarder
    8.8.4.4;
    };
    dnssec-validation auto;  # Enables DNSSEC validation
    listen-on { 172.16.0.10; };  # Listens only on the server's IP address
    allow-transfer { none; };  # Disables zone transfers by default
    };

Then, I created a DNS zone for the domain `dns.ftp.com`. To do this, I modified the local configuration file:

    sudo nano /etc/bind/named.conf.local

And added the following configuration:

    zone "dns.ftp.com" {
    type master;
    file "/etc/bind/db.dns.ftp.com";
    };

After that, I created the zone file by copying the base zone file:

    sudo cp /etc/bind/db.local /etc/bind/db.dns.ftp.com

Then, I modified the file:

    sudo nano /etc/bind/db.dns.ftp.com

And added the following content:

    ; BIND data file for dns.ftp.com
    
    ;
    
    $TTL  604800
    
    @  IN  SOA  ns1.dns.ftp.com. admin.dns.ftp.com. (
    
    2023101001  ; Serial
    
    604800  ; Refresh
    
    86400  ; Retry
    
    2419200  ; Expire
    
    604800 )  ; Negative Cache TTL
    
    ;
    
    @  IN  NS  ns1.dns.ftp.com.
    
    @  IN  A  172.16.0.10  ; DNS server IP address
    
    ns1  IN  A  172.16.0.10  ; DNS server IP address

I saved the file and performed tests to verify the syntax of the zone and configuration files:

    sudo named-checkconf
    
    sudo named-checkzone dns.ftp.com /etc/bind/db.dns.ftp.com

To apply the changes, I restarted the DNS server and enabled it to start automatically:

    sudo systemctl restart bind9
    
    sudo systemctl enable bind9

Next, I configured the DNS client on the second machine by modifying the `resolv.conf` file:

    sudo nano /etc/resolv.conf

And added the following line:

    nameserver 172.16.0.10

Finally, I configured my network card on my Windows PC to connect to the DNS server `172.16.0.10`.

For the final test, I executed the following command on the first machine (the one with the DNS server):

    nslookup dns.ftp.com 172.16.0.10

Successfully.

On the second machine, I ran the same command, which also worked.

# 6. SFTP Server Connection Test

First, I pinged `dns.ftp.com` like this:

    ping dns.ftp.com

Then, I ran:

    nslookup dns.ftp.com 172.16.0.10

To verify the connection.

Next, I connected via SFTP with:

    sftp laplateforme@dns.ftp.com

I was prompted for the password, and the connection was successful.

# 7. Additional Security Settings

On the client machine, I modified the SSH configuration file:

    nano /etc/ssh/sshd_config

And added the following commands:

    Port 6500
    
    AllowUsers laplateforme
    
    PermitRootLogin no