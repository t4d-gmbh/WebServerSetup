# Ansible Role: cudaSupport


## Role Variables

- `scripts_repo_url`:
- `scripts_repo_branch`:
- `scripts_repo_dest`:

...

## Role: TigerVNC Server Setup

### Overview

This Ansible role automates the installation and configuration of the TigerVNC server on an Ubuntu machine. It is designed to set up a graphical desktop environment that allows users to connect remotely via VNC (Virtual Network Computing). The role ensures that the specified user can access the machine's desktop environment securely and efficiently.

### Features

- **Install TigerVNC Server**: Installs the `tigervnc-standalone-server` package, enabling VNC functionality on the machine.
- **Install Desktop Environment**: Installs the XFCE desktop environment along with necessary components to provide a lightweight and user-friendly interface.
- **User Configuration**: Creates a VNC password for the specified user, ensuring secure access to the VNC session.
- **Startup Script**: Configures a startup script that initializes the XFCE desktop environment when the VNC server starts.
- **Systemd Service**: Sets up a systemd service to manage the VNC server, ensuring it starts automatically on boot and can be easily controlled.
- **Customizable**: Allows customization of the VNC password and the username through variables, making it adaptable to different environments.

### Usage

To use this role, define the following variables in your playbook or inventory:

- `new_username`: The username of the user who will connect via VNC.
- `vault_vnc_password`: The password for the VNC session (should be stored in a vault).

Run the role against your target Ubuntu machine to set up the TigerVNC server and enable remote desktop access for the specified user.
