> [!NOTE]
> Create a SSH tunnel to an offline server, and expose that tunnel as a local port

## On the online server, set up an SSH tunnel to the offline server:

   ```
   ssh -L 0.0.0.0:3000:192.168.68.5:3000 user@192.168.68.5
      ssh -L 0.0.0.0:3390:192.168.68.6:3389 SEC\SA.Robert.Shaw@192.168.68.6
   ```

> [!NOTE]
>    This command creates a tunnel from any network interface on the online server's to port 3000 to the offline server's 192.168.68.5:3000.

## Now, on the online server, set up port forwarding to expose the tunneled port to its public interface:

 ### For iptables:
   ```
   iptables -t nat -A PREROUTING -p tcp --dport 3000 -j DNAT --to-destination 127.0.0.1:3000
   iptables -A FORWARD -p tcp -d 127.0.0.1 --dport 3000 -j ACCEPT
   ```
   
 ### Make sure the firewall on the online server allows incoming connections on port 3000:
   ```
   iptables -A INPUT -p tcp --dport 3000 -j ACCEPT
   ```

 ### Now, when you access port 3000 from any network interface, it should forward to the offline server's docker container at 192.168.68.5:3000.

## Set the SSH Tunnel to start automatically

> [!NOTE]
> To set up the SSH tunnel to run automatically on your machine, you can use a systemd service. This method works well on Arch Linux and most modern Linux distributions. Here's a step-by-step guide:

 ### Create a systemd service file:

   ```
   sudo micro /etc/systemd/system/ssh-tunnel.service
   ```

 ### Add the following content to the file:

   ```
[Unit]
Description=SSH tunnels to offline servers
After=network.target

[Service]
ExecStart=/bin/bash -c '\
    /usr/bin/ssh -NT -o ServerAliveInterval=60 -o ExitOnForwardFailure=yes \
      -L 0.0.0.0:3000:192.168.68.5:3000 \
      -L 0.0.0.0:8898:192.168.68.5:8898 \
      monitoring@192.168.68.5 & \
    /usr/bin/ssh -NT -o ServerAliveInterval=60 -o ExitOnForwardFailure=yes \
      -L 0.0.0.0:3390:192.168.68.6:3389 \
      SEP\monitoring2@192.168.68.6 & \
    wait \
'
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
   ```
```
[Unit]
Description=SSH tunnels to offline servers
After=network.target

[Service]
Environment=DOMAIN_USER=SEC\SA.Robert.Shaw
ExecStart=/usr/local/bin/start-ssh-tunnels.sh
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

> [!NOTE]
>    Replace "monitoring" with the actual username you use to connect to the offline server.

 ### Set up SSH key-based authentication if you haven't already:   
   ```
   ssh-keygen -t rsa -b 4096
   ssh-copy-id user@192.168.68.5
   ```

> [!NOTE]
>    When you generate the keygen, just enter through the filename and password options leaving them blank
>    
>    This allows the SSH connection to be made without manual password entry.

 ### Start and enable the service:

   ```
   sudo systemctl enable --now ssh-tunnel.service
   ```

 ### Check the status of the service:

   ```
   sudo systemctl status ssh-tunnel.service
   sudo journalctl -u ssh-tunnel.service
   ```
   

 ### If you need to stop the service at any time:

   ```
   sudo systemctl stop ssh-tunnel.service
   ```

This setup will automatically start the SSH tunnel when the system boots and restart it if it fails for any reason.

A few additional tips:

- The `ServerAliveInterval=60` option sends a keep-alive signal every 60 seconds to prevent the connection from timing out.
- The `ExitOnForwardFailure=yes` option ensures the service stops if the port forwarding fails, allowing systemd to restart it.
- You may want to add `RemoveExistingFiles` to the `[Service]` section if you want to ensure only one instance of the tunnel is running.

Remember to adjust your firewall settings both on the Proxmox host and the guest VM to allow this connection permanently.