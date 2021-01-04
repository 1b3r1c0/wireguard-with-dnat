# wireguard-with-dnat

## VNC & SSH via Hub & Spoke Wireguard VPN

The purpose of this project is to set up a Docker container for Wireguard VPN services on a publicly available VPS (Linode) to be used in a Hub-and-Spoke VPN scenario that will facilitate SSH & VNC connections from a laptop to a home-server. While the VPS must have a static public IP, the laptop & home-server can have dynamically assigned IP addresses (e.g. coffee shop Wi-Fi & residential ISP).

#### Additional objectives/constraints
- Provide both VNC & SSH connectivity from the laptop to the home server
- Use a Docker container on the VPS for providing Wireguard service
- On the home server, VNC should only listen on 127.0.0.1; the VNC server should be started like this: `vncserver -localhost yes`

**As a best practice**, in 'wg0.conf', the public key of the local wireguard device should always be saved unerneath the private key, commented out of course.
        
-------------------------
## Configure the home server

#### Allow the kernel to route external traffic to `lo0`

This is required because the kernel performs an RPF check on traffic destined to 127.0.0.0/8 and won't perform route lookups on traffic not sourced from itself that's destined to its loopback interface 

Note that `wg0` is wireguard's tunnel interface

	sysctl -w net.ipv4.conf.wg0.route_localnet=1

Verify (the ouput of this command should be "1")

	cat /proc/sys/net/ipv4/conf/wg0/route_localnet 

#### Start a non-root user shell on the home server

#### Set the VNC server pasword for the non-root user

    limited@home-server:~$ vncpasswd
    -----------------------------
    Password:
    Verify:
    Would you like to enter a view-only password (y/n)? n

#### Start VNC server as a non-root user

**Note:** Make note of the number after the ":" so you know which VNC port to use. For example, ":2" means connect to port 5902 (5900 + number after the colon)

    vncserver :1 -localhost yes
    (or)
    vncserver -localhost yes
    -----------------------------
    Cleaning stale pidfile '/home/limited/.vnc/home-server.local:2.pid'!
    . . .

**Note:** A Tresorit window popped open when I entered that vncserver command ...

#### Verify VNC is only listening on lo0

	netstat -pentua |grep 590
	-----------------------------
	tcp ... 127.0.0.1:5901 ... LISTEN ... 1092/Xtigervnc      
	. . .

## Configure the VPS

-----
### Edit the Docker compose file

I'm using the wireguard container provided at ghcr.io/linuxserver/wireguard

The following variables in the 'environment:' section must be configured

Set the timezone to EST

    - TZ=America/New_York

Set the FQDN of the hub/VPS

    - SERVERURL=<fqdn-vps> # or 'auto'

List the names of each peer separated with a comma

    - PEERS=home-server,latop

Let each peer set their own DNS server

    - PEERDNS=auto

IP addresses from the 'Internal' subnet are applied to the 'wg0' interfaces (10.13.13/24 is Wireguard's default)

    - INTERNAL_SUBNET=10.13.13.0

The value for `ALLOWEDIPS` in the docker compose file is used in the auto-generated `wg0.conf` files. Since we'll be modifying these files before applying them, the value specified here is not important. Also, it isn't possible to correctly configure each `wg0.conf` file just using this single environmental variable so we'll just use this as a "base" and correct it later:

    - ALLOWEDIPS=10.13.13.1/32,10.13.13.2/32,10.13.13.3/32

This is where the auto-generated `wg0.conf` files will be saved on the Docker host

    volumes:
    - /bu/cv/wireguard/config:/config

This port must be forwarded from the Docker host's public interface to the Wireguard container. I'm just using Wireguard's default port number here. 

    ports:
    - 51820:51820/udp

(The complete docker compose service config is saved in this GitHub respository)

-----
### Start up the container

    docker-compose up -d

-----
### Copy the client configs

On its first run, the container auto-generates all of the `wg0.conf` files for the hub & spokes. These files include the keys.

Copy the content from these files. Most of it will be used in the final `wg0.conf` files.

	cat /bu/cv/wireguard/config/peer_home-server/peer_home-server.conf 
	cat /bu/cv/wireguard/config/peer_laptop/peer_laptop.conf 

-----
### Modify the hub's wg0.conf file

- Add the public key, commented out (my own best practice)
- Set the allowed IPs to each peer's /32

`````
	vim /bu/cv/wireguard/config/wg0.conf
	-------------------------
        [Interface]
        . . .
    +   # PublicKey = ejEV2Qj6PejBYTILEjMZwVDbY5Xgu3ZqMkEFneXB8kE=
        . . .
        
        [Peer]
        # peer_home-server
    >   AllowedIPs = 10.13.13.2/32
    +   PersistentKeepalive = 25
        . . .
  
        [Peer]
        # peer_laptop
    >   AllowedIPs = 10.13.13.3/32
    +   PersistentKeepalive = 25
        . . .    
`````


(complete `wg0.conf` files for the hub and each spoke are saved in this GitHub repository)

-----
### Apply the changes and verify

Restart the container to apply the changes made in the hub's `wg0.conf` file
                                                                                                             
	docker-compose restart wireguard

Verify by starting a shell in the container ...

	docker exec -it wireguard /bin/bash

... and then from the container's shell


	root@c371c037ff77:/# wg show wg0 allowed-ips
	-----------------------
	mzJ+PfFwnhjmyZSQV89B37CgsxOKjrUvfVL2W98H/RQ=	10.13.13.2/32
	/xmvDU14tKay/3ciz1w3fd8rW3F7XJsJ71DYcOphADE=	10.13.13.3/32

**Note:** In the container, peer's will only be identified by their public keys
    
-------------------------
## Modify the laptop's wg0.conf file

- Add keepalive config
- Verify the Interface Address corresponds to the correct spoke name
- Add the public key, commented out (my own best practice)
- Remove/comment-out any DNS config
- Set the allowed IPs to the entire internal subnet

For example

        [Interface]
        Address = 10.13.13.3
        . . .
    +	# PublicKey = /xmvDU14tKay/3ciz1w3fd8rW3F7XJsJ71DYcOphADE=
        . . .
    >	# DNS = 10.13.13.1
    
        [Peer]
        . . .
    >   AllowedIPs = 10.13.13.0/24
    +   PersistentKeepalive = 25

(the complete `wg0.conf` is saved in this GitHub repository)

-------------------------
## Modify the home-server's wg0.conf file

- Perform the same modifications as the laptop's `wg0.conf` file
- Configure DNAT in iptables

For example

        [Interface]
        Address = 10.13.13.2
        . . .
    +	# PublicKey = /xmvDU14tKay/3ciz1w3fd8rW3F7XJsJ71DYcOphADE=
        . . .
    >	# DNS = 10.13.13.1
    +   Postup = iptables -t nat -A PREROUTING -i wg0 -p tcp -m tcp --dport 22 -j DNAT --to-destination 127.0.0.1
    +   PostDown = iptables -t nat -D PREROUTING -i wg0 -p tcp -m tcp --dport 22 -j DNAT --to-destination 127.0.0.1
    
        [Peer]
        . . .
    >   AllowedIPs = 10.13.13.0/24
    +   PersistentKeepalive = 25

(the complete `wg0.conf` is saved in this GitHub repository)

-------------------------
## Configure wireguard on the spokes (requires root)

Debian Bullseye includes wireguard in the main repository

    apt install wireguard wireguard-tools

Copy the content for each of the spoke wg0.conf files here

    vim /etc/wireguard/wg0.conf

Tunnels Up

	wg-quick up wg0

#### After performing the above on all spokes, verify from laptop to home server (10.13.13.2)
Ping the hub

    ping 10.13.13.1

Ping the home server

    ping 10.13.13.2

Attempt SSH

    ssh home-server-user@10.13.13.2

Attempt VNC

    Applications >  Internet > TigerVNC Viewer
    
    VNC Server: 10.13.13.2:5901

--------------------
## Troubleshoot IPTables

Display iptables' rules

	iptables -L -t nat -v --line-numbers

Delete iptables rule number 2 in the nat table's prerouting chain

	iptables -t nat -D PREROUTING 2 

Log traffic destined to TCP/5901 in the nat & filter tables in a few different chains (in each table)

	iptables -t nat -A PREROUTING -m limit --limit 1/sec --limit-burst 10 -p tcp -m tcp --dport 5901 -j LOG --log-prefix "nat-PREROUTING: "
	
	iptables -t nat -A INPUT -m limit --limit 1/sec --limit-burst 10 -p tcp -m tcp --dport 5901 -j LOG --log-prefix "nat-INPUT: "
	
	iptables -t nat -A POSTROUTING -m limit --limit 1/sec --limit-burst 10 -p tcp -m tcp --dport 5901 -j LOG --log-prefix "nat-POSTROUTING: "
	
	iptables -t nat -A OUTPUT -m limit --limit 1/sec --limit-burst 10 -p tcp -m tcp --dport 5901 -j LOG --log-prefix "nat-OUTPUT: "
	
	iptables -t filter -A INPUT -m limit --limit 1/sec --limit-burst 10 -p tcp -m tcp --dport 5901 -j LOG --log-prefix "filter-INPUT: "
	
	iptables -t filter -A FORWARD -m limit --limit 1/sec --limit-burst 10 -p tcp -m tcp --dport 5901 -j LOG --log-prefix "filter-FORWARD: "
	
	iptables -t filter -A OUTPUT -m limit --limit 1/sec --limit-burst 10 -p tcp -m tcp --dport 5901 -j LOG --log-prefix "filter-OUTPUT: "
