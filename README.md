# Make the socks proxy systemwide using tun2socks using a tun interface

I am writing this because I struggled to find an **up-to date** tutorial that explains how we could make a socks proxy system wide like we see in android or some programs for windows like netmod.
[Disclaimer : I am not very proffesional in networks so you might see some stupid steps that I will of course change/improve whenever I see or learn anything new.]

To follow along, you have to get tun2socks binary from [here](https://github.com/xjasonlyu/tun2socks).

Let's consider that we already have an existing socks5 tunnel that is running on our machine waiting for any data to tunnel to the remote server, In my case : I use v2ray vmess + websocket tunnel. 

First of all, let us explain the process in a simple way, see ... the goal we want to achieve is to make a socks proxy system wide meaning that any data we want to get from the internet should be routed through that tunnel heading to the remote server and then the server will handle this data and reply with responses it got, but the catch here is that the socks tunnel itself is one of the programs that are running on our machine and do want to get data from the internet (which is the connection to the server itself) so when we even try to make the tunnel system wide, it will loop around itself, let us use an example.
normal :
```
          the socks tunnel as any normal program
             wants to access the remote server
	          /\                            \ \
	         /  \                            \ \
	         / /                             \  /
	        / /                               \/
I am the process in charge of          the system networking 
forwarding data to socks tunnel              backend
	           /\                             / /
	          /  \                           / /
	           \ \                          / /
	            \ \                        / /
	             \ \                      \  /
	              \ \                      \/
	           oh wait! there is a system wide proxy,
	              I will forward all data to it!

```
So we have to manually add a rule to make the system use the proxy for any internet task rather than the socks tunnel.


## Getting interfaces ready:

start by creating a new tun interface and assign an IP to it :-
```
sudo ip tuntap add mode tun dev tun0 && sudo ip addr add 198.18.0.1/15 dev tun0
```
bring interface up :-
```
sudo ip link set tun0 up
```
Then, we have to delete the default routing rule :-
```
sudo ip route del default
```

## starting systemwide configuration :

Let's consider that I use *wlan0* to connect to the wifi which has a gateway of *10.0.0.2* and a fixed IP address of *10.0.0.100*
```
ip route add default via 198.18.0.1 dev tun0 metric 1
ip route add default via 10.0.0.2 dev wlan0 metric 10
```
Here we told the system to follow two default routing rules to make the tun0 interface (which we will use to tunnel all of the data to socks proxy) to have a higher priority in the system.

Then we have to set another 2 rules in the routing table to make the system ignore the tun interface when it tries to 

 1. send data to the remote server. (you have to know the ip of the remote server let it be *X.X.X.X*)
 2. send a dns query to any dns server (*8.8.8.8*) in our case.
```bash
sudo ip route add X.X.X.X via 10.0.0.2 dev wlan0 src 10.0.0.100
sudo ip route add 8.8.8.8 via 10.0.0.2 dev wlan0 src 10.0.0.100
```
These 2 commands tell the system that whenever you want to access *8.8.8.8* or *X.X.X.X* to use gateway (the **via** option) that has IP address of *10.0.0.2* (my home router) and use the device (the **dev** option) wlan0 (my wifi interface) finally make sure that it sends the data from the local machine's IP address of *10.0.0.100* (the **src** option)
note that : **src** seems to be optional... of course we will use our IP address while we send data using wlan0 interface!!
## tun2socks:
Last but not least, we have to start the tun2socks program which is responsible for taking all the data forwarded to the tun interface and forward it to the socks proxy, considering we are running the socks proxy on port 10808.
```bash
 ./tun2socks-linux-amd64 -device tun0 -proxy socks5://127.0.0.1:10808 -interface wlan0
 ```
 ## Extra :
 in case smth is not working, make sure to set these options to make the interfaces talk to each other
 ```bash
 sysctl net.ipv4.conf.all.rp_filter=0
sysctl net.ipv4.conf.eth0.rp_filter=0
```
 
