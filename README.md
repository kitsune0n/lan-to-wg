# Dockerized WireGuard Gateway in local network

This project creates a Docker container that functions as a gateway on the local network, utilizing a macvlan network to obtain a dedicated IP address within your LAN, and routing all traffic from devices using it as their gateway through a WireGuard VPN tunnel.

## Description

The container uses the `linuxserver/wireguard` image and is configured to operate on two networks:

1. **`docker_bridge_net`:** The standard Docker bridge network for internal communication.
2. **`host_lan_net`:** A `macvlan` network connected to the host's physical network interface, allowing the container to have an IP address on your local network.

All traffic passing through the gateway is routed through the WireGuard VPN.

<details>
  <summary><b>Why Use This? (Benefits)</b></summary>

  This project offers a powerful and flexible alternative to running VPN clients on individual devices or routers, providing:

  *   **Faster Speeds:** Offloads VPN processing from your router to a more powerful machine, resulting in significantly faster VPN speeds. Ideal if your router struggles with WireGuard encryption.
  *   **Unlimited Devices:** Connect all your devices through a single VPN connection, bypassing limitations imposed by some VPN providers on the number of simultaneous connections.
  *   **Simplified Management:** No need to install VPN clients on each device. Manage your VPN connection and routing rules centrally from the Docker container.
  *   **Increased Control:**  Choose which devices or even specific traffic goes through the VPN, giving you more control over your network.
  *   **Potentially Avoid ISP Throttling:** Using WireGuard on a dedicated gateway might help mitigate the impact of ISP throttling on VPN traffic.

  **In short, this solution is ideal if you want a faster, more flexible, and centrally managed VPN setup for your home network, especially if your router's VPN performance is lacking or your VPN provider limits device connections.**
</details>

## Requirements

*   Docker
*   Docker Compose

## Installation and Configuration

1. Clone the repository:

    ```bash
    git clone <your_repo_url>
    cd <your_repo_directory>
    ```

2. Copy the `.env.example` file to `.env` and configure the environment variables:

    ```bash
    cp .env.example .env
    nano .env
    ```

    **Description of environment variables:**

    | Variable              | Description                                                                    | Example          |
    | --------------------- | ------------------------------------------------------------------------------ | ---------------- |
    | `HOST_LAN_INTERFACE` | Name of the network interface connected to the host's local network (VLAN).     | `enp2s0`         |
    | `HOST_LAN_SUBNET`    | Subnet of the host's local network.                                             | `192.168.0.0/24`    |
    | `HOST_LAN_IP`        | IP address of the container on the host's local network.                       | `192.168.0.252`     |
    | `DOCKER_BRIDGE_SUBNET` | Subnet of the internal Docker network.                                         | `172.23.0.0/24` |
    | `DOCKER_BRIDGE_IP`    | IP address of the container on the internal Docker network.                    | `172.23.0.2`    |
    | `DOCKER_BRIDGE_GATEWAY` | Gateway of the internal Docker network.                                      | `172.23.0.1`    |

3. **Configure WireGuard:**

    You need to provide your own WireGuard configuration file. This file typically ends in `.conf` and contains your private key, the server's public key, endpoint, etc.

    *   If you already have a WireGuard configuration file from your VPN provider, rename it to `wg0.conf`.
    *   If you are setting up your own WireGuard server, you'll need to generate a configuration file for this client.

    **Important:** Place your `wg0.conf` file in the same directory as your `docker-compose.yaml` file.

    **Example `wg0.conf` structure:**
```ini
[Interface]
PrivateKey = <Your private key>
Address = <Your IP address in the VPN network>/24

[Peer]
PublicKey = <VPN server's public key>
PresharedKey = <Your Pre-shared key (optional)>
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
Endpoint = <VPN server's IP address and port>
```

3. Add the following lines to the `[Interface]` section for proper routing and traffic masquerading:
```ini
PreUp = ip route del default; ip route add default via ${DOCKER_BRIDGE_GATEWAY} dev eth0; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostUp = iptables -t nat -A POSTROUTING -o wg0 -j MASQUERADE
```
<details>
  <summary><b>Click to expand for details on PreUp and PostUp commands</b></summary>

  These commands perform the following actions:

  -   **`PreUp`:**
      -   `ip route del default`: Deletes the default route.
      -   `ip route add default via ${DOCKER_BRIDGE_GATEWAY} dev eth0`: Adds the default route through the Docker bridge network.
      -   `iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE`: Enables masquerading (NAT) for traffic exiting through `eth0`.
  -   **`PostUp`:**
      -   `iptables -t nat -A POSTROUTING -o wg0 -j MASQUERADE`: Enables masquerading (NAT) for traffic exiting through the `wg0` interface (VPN tunnel).

</details>


4. Start the container:

    ```bash
    docker-compose up -d
    ```

## Usage

To use the container as a gateway, configure the `HOST_LAN_IP` (as defined in your `.env` file) as the default gateway on the devices in your local network. This will route their traffic through the WireGuard VPN tunnel. **Assuming you have set `HOST_LAN_IP=192.168.0.252` in your `.env` file, you would configure `192.168.0.252` as the default gateway on your devices.**

**Alternative Routing Methods:**

Besides setting the container as the default gateway for individual devices, you can configure routing rules on your router or on specific network clients:

*   **Router-based Routing:**
    *   You can configure static routes on your router to direct traffic **from specific devices** (based on their IP or MAC addresses) to the `HOST_LAN_IP` of the container (which is `192.168.0.252` in this case). This allows you to selectively route traffic from certain devices through the VPN, while others use the standard internet connection.
    *   Refer to your router's documentation for instructions on how to configure static routes.
*   **Client-based Routing:**
    *   On individual clients (e.g., computers, servers), you can manually add static routes that direct traffic **to specific destination networks** through the `HOST_LAN_IP` of the container (`192.168.0.252`). This is useful for routing traffic to specific services or websites through the VPN.
    *   The commands for adding static routes vary depending on the operating system.
        *   **Linux:**  `sudo ip route add <destination_network> via <HOST_LAN_IP>`
        *   **Windows:** `route add <destination_network> mask <subnet_mask> <HOST_LAN_IP>`
        *   **macOS:** `sudo route -n add -net <destination_network> -gateway <HOST_LAN_IP>`

**Note:**

*   Replace `<destination_network>` and `<subnet_mask>` with the appropriate network and subnet mask you want to route through the VPN.
*   The commands provided for adding static routes are examples. Consult your operating system's documentation for the exact syntax.
<details>
  <summary><b>Important Considerations about Static Routes and DNS Leaks:</b></summary>


*   **Potential DNS Leaks:** When using static routes, especially client-based, your DNS requests might still be sent through your regular internet connection, potentially revealing your browsing activity to your ISP or other parties. This is because static routes typically only affect the routing of IP packets, not the DNS resolution process.
*   **Mitigation:** To minimize the risk of DNS leaks when using static routes:
    *   **Configure your devices to use the DNS servers provided by your VPN provider.** This ensures that your DNS requests are also routed through the VPN tunnel, **provided you have added a static route for these DNS servers to go through the VPN gateway ( `192.168.0.252` in this case).** You can usually find these DNS server addresses in your VPN provider's documentation or client application.
        **Example:** If your VPN provider's DNS server is `1.1.1.1`, you would add a static route for this IP address via the gateway, similar to the examples for destination networks given above. For instance:
         **On Linux:**
         ```bash
         sudo ip route add 1.1.1.1 via 192.168.0.252
         ```
         **On Windows:**
         ```bash
         route add 1.1.1.1 mask 255.255.255.255 192.168.0.252
         ```
         **On macOS:**
         ```bash
         sudo route -n add -host 10.8.0.1 -gateway 192.168.0.252
         ```
    *   **Use DNS Leak Test websites:** After configuring static routes, use websites like `dnsleaktest.com` or `ipleak.net` to check if your DNS requests are leaking.
    *   **Consider using the container as your default gateway:** If possible, setting the container as the default gateway for your devices is the most reliable way to avoid DNS leaks, as it forces all traffic, including DNS requests, through the VPN tunnel
      </details>
## License

This project is licensed under the [MIT License](LICENSE).

## Author

[Stanislav Prokopenko](https://github.com/kitsune0n)
