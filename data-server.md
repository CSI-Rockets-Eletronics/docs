# CSI Rockets Data Server Documentation

## Overview

This repository implements a telemetry data server for rocket testing. It is designed to collect, store, and synchronize telemetry data (such as sensor readings and messages) from multiple devices during rocket test sessions. The server is built with [Bun](https://bun.sh/), [Elysia](https://elysiajs.com/), and [PostgreSQL](https://www.postgresql.org/), and uses [Prisma](https://www.prisma.io/) as its ORM.

---

## Key Concepts

- **Environment:** A logical grouping (e.g., a specific rocket or test stand). For security, use obfuscated names (e.g., `env_abc123`), never real rocket names.
- **Session:** A single test run or experiment, with a start time.
- **Device:** The source of telemetry data (e.g., a sensor or microcontroller).
- **Record:** A timestamped data point from a device.
- **Message:** A timestamped message, often used for commands or events. The messages API functions as a command queue.

---

## System Architecture

- Multiple data servers can be deployed in a mesh setup. For example, one server may run in the Mechanical Engineering lab and another on the Raspberry Pi at the firing station.
- By setting the `PARENT_NODE_URL` environment variable, you can configure a server (like the Raspberry Pi) to periodically relay its data to a parent server (like the MechE lab server). This provides:
  - **Backup:** Data is stored in more than one place.
  - **Live Telemetry:** The parent server can be made accessible on the public internet, so anyone can view live telemetry data remotely.
  - **Redundancy:** If one server goes down, the other still has a copy of the data.

---

## Deployment

### Database Deployment

- The database is PostgreSQL.
- The connection string is set in the `.env` file:
  ```
  DATABASE_URL="postgresql://<user>@localhost:5432/rockets-data?schema=public"
  ```
- The schema is managed by Prisma (`prisma/schema.prisma`).
- Migrations are in `prisma/migrations`.

#### On Raspberry Pi or Linux Server

- The `setup-pi/setup.sh` script automates:
  - Installing PostgreSQL 15
  - Configuring PostgreSQL for remote access
  - Setting up the database user and password
  - Installing Bun and Node.js
  - Installing dependencies and generating the Prisma client
  - Deploying the service as a systemd service

#### With Docker

- The repository includes a `Dockerfile` and `compose.yaml` for containerized deployment.
- The `compose.yaml` file defines two services:
  - `app`: the telemetry server
  - `postgres`: the PostgreSQL database

---

## Server Setup on Raspberry Pi

### 1. SSH into the Raspberry Pi

On your Mac:

```sh
ssh pi@<raspberry-pi-ip-address>
```

Replace `<raspberry-pi-ip-address>` with your Pi’s actual IP.

### 2. Clone the Repository

```sh
git clone https://github.com/<your-org-or-username>/data-server-node.git
cd data-server-node
```

### 3. Run the Setup Script

```sh
cd setup-pi
chmod +x setup.sh
./setup.sh
```

### 4. Configure Environment Variables

Edit the `.env` file in the project root:

```sh
vim ../.env
```

Make sure `DATABASE_URL` and other settings are correct.

### 5. Start, Stop, and Restart the Server Daemon

The server runs as a systemd service called `data-server-node`:

- Start:
  ```sh
  sudo systemctl start data-server-node
  ```
- Stop:
  ```sh
  sudo systemctl stop data-server-node
  ```
- Restart:
  ```sh
  sudo systemctl restart data-server-node
  ```
- Status:
  ```sh
  sudo systemctl status data-server-node
  ```
- View logs:
  ```sh
  journalctl -u data-server-node -f
  ```

---

## Vim Basics (for Editing Files)

### Install Vim

```sh
sudo apt update
sudo apt install vim
```

### Usage

- Open a file: `vim filename`
- Insert mode: Press `i`
- Save: Press `Esc`, then type `:w`
- Quit: Press `Esc`, then type `:q`
- Save & quit: `:wq`
- Quit without saving: `:q!`
- Search: `/pattern`, then `n` for next match

---

## SSH Key Setup and GitHub Integration

### 1. Generate SSH Key Pair

```sh
ssh-keygen -t ed25519 -C "your_email@example.com"
```

### 2. Add Public Key to Raspberry Pi

```sh
ssh-copy-id pi@fs-pi.local
```

Or manually copy `~/.ssh/id_ed25519.pub` to the Pi’s `~/.ssh/authorized_keys`.

### 3. Configure Hostnames During OS Install

- Set hostname to `fs-pi` for the firing station Pi.
- Set hostname to `rocket-pi` for the rocket Pi.

### 4. Configure SSH Aliases in `~/.ssh/config`

```sh
vim ~/.ssh/config
```

Example:

```
Host fs
    HostName fs-pi.local
    User pi

Host rocket
    HostName rocket-pi.local
    User pi
```

Now you can SSH with `ssh fs` or `ssh rocket`.

### 5. Set Up SSH Agent Forwarding

Add to your host config:

```
Host fs
    HostName fs-pi.local
    User pi
    ForwardAgent yes
```

Start the agent and add your key:

```sh
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

### 6. Set Up GitHub SSH Keys

- Copy your public key:
  ```sh
  cat ~/.ssh/id_ed25519.pub
  ```
- Go to [GitHub > Settings > SSH and GPG keys](https://github.com/settings/keys) and add your public key.
- Clone via SSH:
  ```sh
  git clone git@github.com:your-username/data-server-node.git
  ```

---

## API Usage with `curl`

### Environments

- Use obfuscated names (e.g., `env_abc123`) for security.

### Creating a Session

```sh
curl -X POST http://localhost:3000/rocketsdata2/sessions/create \
  -H "Content-Type: application/json" \
  -d '{"environmentKey":"env_abc123"}'
```

### Adding Records

#### Single Record

```sh
curl -X POST http://localhost:3000/rocketsdata2/records \
  -H "Content-Type: application/json" \
  -d '{"environmentKey":"env_abc123","device":"main","ts":1713200000000,"data":{"pressure":123,"temp":45.6}}'
```

#### Batch Records (same device)

```sh
curl -X POST http://localhost:3000/rocketsdata2/records/batch \
  -H "Content-Type: application/json" \
  -d '{
    "environmentKey":"env_abc123",
    "device":"main",
    "records":[
      {"ts":1713200000001,"data":{"pressure":124}},
      {"ts":1713200000002,"data":{"pressure":125}}
    ]
  }'
```

#### Batch Records (multiple devices/environments)

```sh
curl -X POST http://localhost:3000/rocketsdata2/records/batchGlobal \
  -H "Content-Type: application/json" \
  -d '{
    "records":[
      {"environmentKey":"env_abc123","device":"main","ts":1713200000003,"data":{"pressure":126}},
      {"environmentKey":"env_xyz789","device":"backup","ts":1713200000004,"data":{"pressure":127}}
    ]
  }'
```

### Querying Records

#### List Records (JSON)

```sh
curl "http://localhost:3000/rocketsdata2/records?environmentKey=env_abc123&device=main"
```

#### Pagination Options

- `startTs`: Only records after this timestamp (inclusive)
- `endTs`: Only records before this timestamp (inclusive)
- `take`: Max number of records to return

Example (get 10 records after a timestamp):

```sh
curl "http://localhost:3000/rocketsdata2/records?environmentKey=env_abc123&device=main&startTs=1713200000000&take=10"
```

#### Export as CSV

```sh
curl "http://localhost:3000/rocketsdata2/export/env_abc123/latest/main/records" -o records.csv
```

- Replace `latest` with a session name, or use `all` for all sessions.
- Add `startTs`/`endTs` as query params for time filtering.

### Messages API (Command Queue)

#### Add a Message (enqueue a command)

```sh
curl -X POST http://localhost:3000/rocketsdata2/messages \
  -H "Content-Type: application/json" \
  -d '{"environmentKey":"env_abc123","device":"main","data":{"command":"arm"}}'
```

#### Poll the Next Message

```sh
curl "http://localhost:3000/rocketsdata2/messages/next?environmentKey=env_abc123&device=main"
```

- Returns the next message (if any) for the device.
- Use `afterTs` to get messages after a specific timestamp.

### Sessions

#### List Sessions

```sh
curl "http://localhost:3000/rocketsdata2/sessions?environmentKey=env_abc123"
```

#### Get Current Session

```sh
curl "http://localhost:3000/rocketsdata2/sessions/current?environmentKey=env_abc123"
```

### Multi-Device Query

Get the latest record from several devices:

```sh
curl "http://localhost:3000/rocketsdata2/records/multiDevice?environmentKey=env_abc123&devices=main,backup"
```

---

## Ports and Nginx

### Local Development

- The server listens on the port specified by the `PORT` variable in your `.env` file (default: 3000).
- Access the API at `http://localhost:3000`.

### Production (MechE Lab Server)

- The data server may run on any available port.
- **Nginx** is used as a reverse proxy:
  - Nginx listens on standard web ports (80/443).
  - Nginx forwards requests to the internal port where the data server is running.
  - The API is accessible at `https://csiwiki.me.columbia.edu/rocketsdata2` regardless of the internal port.

#### What is Nginx?

- **Nginx** is a high-performance web server and reverse proxy.
- It acts as a gateway between the public internet and your data server.
- Handles HTTPS (SSL/TLS) for secure connections.
- Forwards requests from a public URL to your app’s internal port.
- Users never see or need to know the internal port.

---

## Example `.env` File

```properties
DATABASE_URL="postgresql://<user>@localhost:5432/rockets-data?schema=public"
PORT="3000"
NODE_NAME="Local"
# PARENT_NODE_URL="https://csiwiki.me.columbia.edu/rocketsdata2"

TEST_NODE_URL="http://localhost:3000"
# TEST_NODE_URL="https://csiwiki.me.columbia.edu/rocketsdata2"
```

---

## Summary

- Use obfuscated environment names for security.
- Multiple servers can sync data for redundancy and live telemetry.
- Use systemd to manage the server as a daemon.
- Use Vim for editing files on the Pi.
- Use SSH keys and agent forwarding for secure, convenient access and GitHub integration.
- Use Nginx to expose the server at a public URL.
- Interact with the API using `curl` for all major operations: sessions, records, messages, and exports.

For more details, see the codebase and comments in each file.
