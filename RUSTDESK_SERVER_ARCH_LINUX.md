# Self-Hosted RustDesk Server and Client Setup on Arch Linux

This guide provides a comprehensive walkthrough for setting up a self-hosted RustDesk server on a machine running Arch Linux or its derivatives (like Omarchy Linux), and for configuring the clients to connect to it. This setup is ideal for creating a private, local remote desktop service.

## Prerequisites

Before you begin, ensure you have the following:

1.  **A machine running Arch or an Arch-based Linux distribution.** This will be your server.
2.  **Docker and Docker Compose installed.** Docker is used to containerize and run the RustDesk server applications.
    *   You can check if Docker is installed by running: `docker --version`
    *   You can check if Docker Compose is installed by running: `docker-compose --version`
3.  **Basic knowledge of the command line and your server's local IP address.** You can find your server's local IP address by running: `ip addr show`

## Step 1: Create the Project Directory

First, create a directory where you will store your RustDesk server configuration files.

```bash
mkdir -p ~/Work/Docker/rustdesk
cd ~/Work/Docker/rustdesk
```

## Step 2: Create the Docker Compose File

Next, create a `docker-compose.yml` file in this directory. This file will define the RustDesk server services (`hbbs` and `hbbr`).

```bash
touch docker-compose.yml
```

Now, open the `docker-compose.yml` file and add the following content. **Remember to replace `your_local_ip` with your server's actual local IP address.**

```yaml
services:
  hbbs:
    image: rustdesk/rustdesk-server:latest
    command: hbbs -r your_local_ip
    volumes:
      - ./data:/root
    network_mode: "host"
    depends_on:
      - hbbr
    restart: unless-stopped

  hbbr:
    image: rustdesk/rustdesk-server:latest
    command: hbbr
    volumes:
      - ./data:/root
    network_mode: "host"
    restart: unless-stopped
```

### Explanation of `docker-compose.yml`

*   `services`: This section defines the different applications that will be run.
*   `hbbs`: This is the RustDesk ID server. It's responsible for managing client connections.
    *   `image`: Specifies the Docker image to use (`rustdesk/rustdesk-server`).
    *   `command: hbbs -r your_local_ip`: This is the crucial command that starts the ID server. The `-r` flag tells the ID server to advertise the relay server's address as `your_local_ip`.
    *   `volumes: - ./data:/root`: This mounts the `data` directory from your host machine into the `/root` directory inside the container. This is where the server will store its persistent data, including the cryptographic key.
    *   `network_mode: "host"`: This makes the container use your host machine's network stack directly. It's simpler for this local setup than creating a bridged network.
    *   `restart: unless-stopped`: This ensures that the server will automatically restart if it crashes, unless you manually stop it.
*   `hbbr`: This is the RustDesk relay server. It handles the actual remote desktop traffic between clients.

## Step 3: Start the RustDesk Server

Now, from within your `~/Work/Docker/rustdesk` directory, start the server using Docker Compose:

```bash
docker-compose up -d
```

This command will download the Docker image (if you don't have it already) and start the `hbbs` and `hbbr` services in the background.

A `data` directory will be created in your project folder. Inside this directory, you will find a file named `id_ed25519.pub`. This file contains your public key, which you will need to configure your clients.

## Step 4: Configure the Firewall

If you have a firewall running, you need to allow traffic on the ports that RustDesk uses. For `ufw` (Uncomplicated Firewall), you can run the following commands:

```bash
sudo ufw allow 21115:21119/tcp
sudo ufw allow 21115:21119/udp
sudo ufw reload
```

## Step 5: Configure Your RustDesk Clients

To connect to your self-hosted server, you must configure each client device, including the one running on the server machine itself.

### A. Host Client Configuration (On the Server Machine)

The RustDesk client running on the same machine as the Docker server needs to know how to connect to it.

1.  **Open the RustDesk client** on your server machine.
2.  Navigate to the **Settings** menu (usually represented by three dots `...`).
3.  Select **"ID/Relay Server"**.
4.  In the **ID Server** field, enter the local IP address of your server (the same `your_local_ip` you used in the `docker-compose.yml` file).
5.  The **Relay Server** field can be left blank.
6.  In the **Key** field, you must paste the public key from your server. You can display the key by running this command in your terminal:
    ```bash
    cat ~/Work/Docker/rustdesk/data/id_ed25519.pub
    ```
7.  Click **Apply**. After a moment, the status at the bottom of the client should change from "Connecting..." to **"Ready"**.

### B. Remote Client Configuration (e.g., iOS device, another laptop)

This is the device you will use to connect *to* your server.

1.  Ensure the remote device is connected to the **same local network** as your server.
2.  **Open the RustDesk client**.
3.  Go to **Settings -> ID/Relay Server**.
4.  In the **ID Server** field, enter the local IP address of your server machine (`your_local_ip`).
5.  The **Relay Server** field can be left blank.
6.  In the **Key** field, paste the **exact same public key** you used for the host client.
7.  Return to the main screen. The client should show a **"Ready"** status.
8.  You can now enter the RustDesk ID of your host machine (as shown in its client window) and click "Connect" to start a session.

## Troubleshooting

*   **"failed to connect to 127.0.0.1:21116"**: This error on the client means it's not configured to point to your server. Double-check the "ID/Relay Server" settings on that client to ensure the "ID Server" is set to your server's IP address and not `127.0.0.1`.
*   **"failed to connect via relay server"**: This usually means the clients can reach the ID server (`hbbs`) but not the relay server (`hbbr`). The most common cause is the `-r` parameter in the `docker-compose.yml` file. Make sure it's set to your server's actual local IP address, and then restart the server with `docker-compose up -d --force-recreate`.
*   **Connection Timeouts**: If the connection times out, it's almost always a firewall issue. Ensure your firewall on the server machine is configured to allow traffic on ports `21115-21119` for both TCP and UDP.

That's it! You now have a private, self-hosted RustDesk server for local remote desktop connections.
