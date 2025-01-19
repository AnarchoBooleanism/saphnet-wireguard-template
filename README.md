# saphnet-wireguard-template
Template configuration files for WireGuard interfaces for Saphnet VPN VM, with tutorial

## Setting up a WireGuard connection
To connect two nodes, a server and a client (which may be behind a NAT), you will need to have WireGuard configurations on both ends, started. This guide assumes that you are running a Linux environment that includes wireguard-tools (we are using wg-quick for this guide). In this guide, the server is defined as a WireGuard instance with its own defined IP address and port that other WireGuard instances, which are defined as the clients, connect to. This guide can be used to make multiple WireGuard instances across multiple files.

### Step 1: Deciding keypairs, ports, and IP address pairs
Both the client and server will each need to have a keypair of a private and public key. To create such a keypair, run `wg genkey | tee >(cat >&2) | wg pubkey`, which will, in order, generate a random private key, and a corresponding public key. If a pre-shared key is desired, then run `wg genpsk` to generate one. Again, be sure to have two keypairs and one pre-shared key (if desired), and save them somewhere secure.

You will also need an IP address that each particular node will communicate to the other as. Be sure to pick a local IP address that isn't being used by anything else, and for our purposes, try to use the /32 subnet, to be a whole IP address. The server IP address can be shared across multiple WireGuard instances, but the port cannot be reused across instances, however. You simply need to make sure that each peer has their own defined address.

### Step 2: Creating the server instance configuration file
In `/etc/wireguard` in your server node, create a `.conf` file with any name of your choice, often shown with a name such as `wg0.conf`. The file name does not matter, but please note that this will act as the network interface name (sans `.conf`) that Linux and wg-quick will work with.

In this file, simply copy and fill out the [server instance configuration template file](server-template.conf). The `PrivateKey` entry under `[Interface]` is the server's private key, and the `PrivateKey` entry under `[Peer]` is the client's public key. Be sure to fill out the `Address` and `Port` entries under `[Interface]` with what the server will operate as and listen from, and the `AllowedIPs` entry under `[Peer]` with the IP address that the client will connect as. In addition, any pre-shared key will go into the `PresharedKey` entry under `[Peer]`.

The iptables `iptables -A FORWARD` entries are in the configuration file to allow packets to forwarded in and out of the interface, and the iptables `iptables -t nat -A POSTROUTING -o <NETWORK_INTERFACE> -j MASQUERADE` entry enables Network Access Translation (NAT) for packets going in and out of the external interface. Be sure to replace <NETWORK_INTERFACE> with the name of the network interface that you want to use to interact with outside IP addresses. (`%i` in this instance is a placeholder that is automatically replaced with the WireGuard interface name)

If you want to have specific ports forwarded to an external interface that the server is connected to, be sure to fill out the `iptables` entries (and create more as needed per port) with the external port to forward from and the client IP address and port to forward to. There are individual entries for both TCP and UDP.

Be sure to save the file.

### Step 3: Creating the client instance configuration file
In `/etc/wireguard` in your client node, create a `.conf` file with any name of your choice. Again, the file name does not matter, but please note that this will act as the network interface name that Linux and wg-quick will work with.

In this file, simply copy and fill out the [client instance configuration template file](client-template.conf). The `PrivateKey` entry under `[Interface]` is the client's private key, and the `PrivateKey` entry under `[Peer]` is the server's public key. Be sure to fill out the `Endpoint` entry under `[Interface]` with the IP address and port for the server, and the `AllowedIPs` entry under `[Peer]` with the IP address range(s) that the client will use WireGuard through the server to connect to. In addition, any pre-shared key will go into the `PresharedKey` entry under `[Peer]`.

Again, be sure to save the file.

### Step 4: Activating the WireGuard configurations and connecting the nodes
Now that you have the configuration files on both ends filled out, you are now able to use wg-quick to activate WireGuard on both ends. On both the client and server, run `wg-quick up <CONFIG_FILE_NAME>` (you will need root) to activate a WireGuard connection. It should work, and from one node, you should be able to ping the IP address of the other node and receive a reply. If it does not work, then refer to the troubleshooting section. 

*Note:* To take down the WireGuard connection, simply run `wg-quick down <CONFIG_FILE_NAME>`.

### Step 5: Creating services to automatically set up WireGuard on startup (optional)
If you want wg-quick to automatically run on startup, you will need to create a systemd service. To do this, simply run `systemctl enable wg-quick@<CONFIG_FILE_NAME>`.

## Troubleshooting

If something is going wrong, be sure to check for these:
- The private/public keys are in the correct spots in the configuration files
- The port works and is not being used for anything else
- The client and server IP addresses correspond to the correct spots in the configuration files
- The client is able to connect to the server (try using ping) outside of WireGuard
- The `AllowedIPs` entry in the client configuration file include all of the IPs you want to connect to (including of the server itself), aren't too broad, or are in conflict with other IP ranges from other interfaces or programs
- `wg show` shows something happening (including handshake information) and `ip route` shows a route for the WireGuard interface
- `tcpdump -i <CONFIG_FILE_NAME>` shows activity when pings happen
- The kernel and other networking tools are in working order
