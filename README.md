# wiregaurd-with-dnat

## VNC & SSH via Hub & Spoke Wireguard VPN

The purpose of this project is to connect a laptop to a homeserver through a Wireguard VPN configured on a VPS

#### To Do
- set keepalives in all peer configs
- move config into the spoke's wg0.conf file:
    - iptables DNAT
    - sysctl route local net = 1
- Put this project on GitHub
    - README
    - VPS Docker compose excerpt
    - wg0.conf files
        - VPS
        - linlap
        - dtat03

#### Additional objectives/constraints
- Provide both VNC & SSH connectivity from the laptop to the home server
- Use a Docker container on the VPS for providing Wireguard service
- On the home server, VNC should only listen on 127.0.0.1; the VNC server should be configured as `vncserver -localhost yes`

**As a best practice**, in 'wg0.conf', the public key of the local wireguard device should always be saved unerneath the private key, commented out of course.
        
-------------------------
## Configure the home server

Allow the kernel to route traffic to lo0

This is required because the kernel performs an RPF check on traffic destined to 127.0.0.0/8 and won't perform route lookups on traffic not sourced from itself

'wg0' is wiregaurd's tunnel interface

	sysctl -w net.ipv4.conf.wg0.route_localnet=1

Verify (output should be "1")

	cat /proc/sys/net/ipv4/conf/wg0/route_localnet 

Start a non-root user shell on the home server

Set the VNC server pasword for the non-root user

    limited@home-server:~$ vncpasswd
    -----------------------------
    Password:
    Verify:
    Would you like to enter a view-only password (y/n)? n

Start VNC server as a non-root user

**Note:** Make note of the number after the ":" so you know which VNC port to use. For example, ":2" means connect to port 5902 (5900 + number after the colon)

    vncserver :1 -localhost yes
    (or)
    vncserver -localhost yes
    -----------------------------
    Cleaning stale pidfile '/home/limited/.vnc/home-server.local:2.pid'!
    . . .
    
**Note:** A Tresorit window popped open when I entered that vncserver command ...
    
Verify VNC is only listening on lo0

	netstat -pentua |grep 590
	-----------------------------
	tcp . . . 127.0.0.1:5901 . . . LISTEN . . . 1092/Xtigervnc      
	. . .

#### Configure DNAT in iptables (root required)
- DNAT to 127.0.0.1, traffic coming in on the wg0 interface, destined to any IP, on destination ports TCP/5900-5910 (leave the dport alone)

	iptables -t nat -A PREROUTING -i wg0 -p tcp -m tcp --dport 5900:5910 -j DNAT --to-destination 127.0.0.1

- DNAT to 127.0.0.1, traffic coming in on the wg0 interface, destined to any IP, on destination port TCP/22 (leave the dport alone)

	iptables -t nat -A PREROUTING -i wg0 -p tcp -m tcp --dport 22 -j DNAT --to-destination 127.0.0.1

Display iptables' rules

	iptables -L -t nat -v --line-numbers

Delete iptables rule number 2 in the nat table's prerouting chain

	iptables -t nat -D PREROUTING 2 
    
-------------------------
## Configure the VPS

-----
### Edit the Docker compose file

I'm using the wireguard container provided at ghcr.io/linuxserver/wireguard

Required varibles in the 'environment:' section

Set the timezone

    - TZ=America/New_York

Server URL

    - SERVERURL=<fqdn-vps> # or 'auto'

Names of peers separated by commas

    - PEERS=home-server,latop

Let each peer set their own DNS server

    - PEERDNS=auto

IPs from this subnet will be applied to the 'wg0' interfaces on each hub and spoke (Just using Wireguard's default here)

    - INTERNAL_SUBNET=10.13.13.0

These values will be used in the auto-generated configs for the peers. Since those configs can be modified before they're applied, this value is not super important. We'll be using split tunneling so I won't specify 0.0.0.0/0. Its not posible to configure everything correctly in this single env var so we'll just start with this and then correct it later:

    - ALLOWEDIPS=10.13.13.1/32,10.13.13.2/32,10.13.13.3/32

This is where the auto-generated configs will be saved and also where wg0.conf for the hub will be saved

    volumes:
    - /bu/cv/wireguard/config:/config

Just using Wireguard's default port number. This will have to be forwarded from the VPS' public interface to the Docker container

    ports:
    - 51820:51820/udp

The complete config is saved in this GitHub respository

-----
### start up the container

    docker-compose up -d

-----
### Copy the client configs

On its first run, the container will auto-generate all of the 'wg0.conf' files for the hub & spoke peers. The conf files will include the keys
 
Copy the content from these files; they'll be used as base configs

	cat /bu/cv/wireguard/config/peer_home-server/peer_home-server.conf 
	cat /bu/cv/wireguard/config/peer_laptop/peer_laptop.conf 

-----
### Modify the hub's wg0.conf file

- **To Do:** Add keepalive config
- Add the public key, commented out (my own best practice)
- Set the allowed IPs to each peer's /32

	    vim /bu/cv/wireguard/config/wg0.conf
	    -------------------------
        [Interface]
        . . .
    +   # PublicKey = ejEV2Qj6PejBYTILEjMZwVDbY5Xgu3ZqMkEFneXB8kE=
        . . .
        
        [Peer]
        # peer_home-server
    >   AllowedIPs = 10.13.13.2/32
    ?   PersistentKeepalive = 25
        . . .
       
        [Peer]
        # peer_laptop
    >   AllowedIPs = 10.13.13.3/32
    ?   PersistentKeepalive = 25
        . . .    
    
The complete config is saved in this GitHub respository

-----
### Apply the changes and verify

Restart the container to apply the changes made in the hub's wg0.conf file
                                                                                                             
	docker-compose restart wireguard

Verify by starting a shell in the container ...

	docker exec -it wireguard /bin/bash

... and then from the container's shell


	root@c371c037ff77:/# wg show wg0 allowed-ips
	-----------------------
	mzJ+PfFwnhjmyZSQV89B37CgsxOKjrUvfVL2W98H/RQ=	10.13.13.2/32
	/xmvDU14tKay/3ciz1w3fd8rW3F7XJsJ71DYcOphADE=	10.13.13.3/32

**Note:** Peer's will only be identified by their public keys
    
-------------------------
## Modify the peer wg0.conf files

In each spoke wg0.conf file
- Add keepalive config
- verify the Interface Address correpsonds to the correct spoke name
- Add the public key, commented out (my own best practice)
- Remove/comment-out any DNS config
- Set the allowed IPs to the entire internal subnet

For example

        vim /path/to/wg0.conf
        ----------------------------
	    [Interface]
	    Address = 10.13.13.2
	    . . .
    +	# PublicKey = mzJ+PfFwnhjmyZSQV89B37CgsxOKjrUvfVL2W98H/RQ=
	    . . .
    >	# DNS = 10.13.13.1

	    [Peer]
	    . . .
    >   AllowedIPs = 10.13.13.0/24
    ?   PersistentKeepalive = 25
    
The complete configs are saved in this GitHub respository

-------------------------
## Configure wireguard on the spokes (requires root)

Debian Bullseye includes wireguard in the main repository

    apt install wireguard wireguard-tools

Copy the content for the spoke wg0.conf files here

	vim /etc/wireguard/wg0.conf

Tunnels Up

	wg-quick up wg0
	
After performing the above on all spokes, verify from laptop to home server (10.13.13.2)
- Ping the hub

    ping 10.13.13.1

- Ping the home server

    ping 10.13.13.2

- Attempt SSH

    ssh home-server-user@10.13.13.2
    
- Attempt VNC

    Applications >  Internet > TigerVNC Viewer
    
    VNC Server: 10.13.13.2:5901
        
--------------------
## Troubleshoot IPTables by logging

Log traffic destined to TCP/5901 in the nat & filter tables in a few different chains (in each table)
 
	iptables -t nat -A PREROUTING -m limit --limit 1/sec --limit-burst 10 -p tcp -m tcp --dport 5901 -j LOG --log-prefix "nat-PREROUTING: "
	
	iptables -t nat -A INPUT -m limit --limit 1/sec --limit-burst 10 -p tcp -m tcp --dport 5901 -j LOG --log-prefix "nat-INPUT: "

	iptables -t nat -A POSTROUTING -m limit --limit 1/sec --limit-burst 10 -p tcp -m tcp --dport 5901 -j LOG --log-prefix "nat-POSTROUTING: "

	iptables -t nat -A OUTPUT -m limit --limit 1/sec --limit-burst 10 -p tcp -m tcp --dport 5901 -j LOG --log-prefix "nat-OUTPUT: "

	iptables -t filter -A INPUT -m limit --limit 1/sec --limit-burst 10 -p tcp -m tcp --dport 5901 -j LOG --log-prefix "filter-INPUT: "

	iptables -t filter -A FORWARD -m limit --limit 1/sec --limit-burst 10 -p tcp -m tcp --dport 5901 -j LOG --log-prefix "filter-FORWARD: "

	iptables -t filter -A OUTPUT -m limit --limit 1/sec --limit-burst 10 -p tcp -m tcp --dport 5901 -j LOG --log-prefix "filter-OUTPUT: "
