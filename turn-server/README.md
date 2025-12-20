# Standalone TURN Relay Server

This directory contains a complete, isolated configuration for a `coturn` TURN server, designed to be run with Docker. This server acts as a media relay for your WebRTC application, improving connection reliability.

### ⚠️ Critical Pre-flight Check

Before you run this server for anyone outside your local network, you **must** update its configuration:

1.  **Open `turnserver.conf`**.
2.  **Find the `external-ip` line**:
    ```
    external-ip=127.0.0.1
    ```
3.  **Change `127.0.0.1`** to the public IP address of the server where you are deploying this container.

### How to Run

1.  Ensure Docker and Docker Compose are installed on your server.
2.  Navigate to this directory (`turn-server`) in your terminal.
3.  Execute the following command:

    ```sh
    # This will build the container if needed and start it in the background.
    docker-compose up -d
    ```

### Server Credentials

The following credentials are required by the frontend application to authenticate with this server. They are defined in `turnserver.conf`.

-   **URL**: `turn:YOUR_PUBLIC_IP:3478`
-   **Username**: `user_claude_123`
-   **Password**: `pass_lkjHn$98a`

Remember to replace `YOUR_PUBLIC_IP` with the actual public IP of your server.
