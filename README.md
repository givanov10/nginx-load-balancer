# Nginx Reverse Proxy & Load Balancer

A practical Linux system administration project demonstrating how to configure **Nginx** as a **Reverse Proxy** and **Load Balancer** in front of multiple backend web services.

## Architecture Overview

Instead of exposing backend applications directly to clients, Nginx sits on standard HTTP port `80` and distributes incoming traffic across two local Python HTTP servers using the **Round Robin** algorithm.

```text
                  ┌──────────────────────┐
                  │  Backend Server 1    │
               ┌─▶│  (127.0.0.1:8001)    │
┌──────────┐   │  └──────────────────────┘
│  Client  │───┼──▶ Nginx (Port 80)
└──────────┘   │  ┌──────────────────────┐
               └─▶│  Backend Server 2    │
                  │  (127.0.0.1:8002)    │
                  └──────────────────────┘
```

## Project Structure

* `app1/index.html` – Static content served by the first backend instance (`Port 8001`).
* `app2/index.html` – Static content served by the second backend instance (`Port 8002`).
* `nginx.conf` – Nginx configuration defining the `upstream` server group and `proxy_pass` routing rules.

## Setup & Usage

### 1. Start the Backend Servers
Launch both lightweight Python HTTP servers in the background:

```bash
cd app1 && python3 -m http.server 8001 &
cd app2 && python3 -m http.server 8002 &
```

### 2. Configure Nginx
Copy the configuration file to Nginx's available sites and enable it via a symbolic link:

```bash
sudo cp nginx.conf /etc/nginx/sites-available/my-balancer
sudo ln -s /etc/nginx/sites-available/my-balancer /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
```

Test the syntax and reload the service:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### 3. Test Load Balancing
Send multiple sequential requests to port `80` to verify traffic distribution:

```bash
for i in {1..4}; do curl http://127.0.0.1; done
```

**Expected Output:**
```text
Hello from server 1
Hello from server 2
Hello from server 1
Hello from server 2
```

## Key Concepts Covered
* Linux process management (`&`, `kill`, `ss`, `pwdx`)
* Network interfaces and port binding (`0.0.0.0` vs `127.0.0.1`)
* Nginx `upstream` blocks and Reverse Proxy configuration
* High Availability (automatic failover when a backend instance goes down)
* Log analysis (`/var/log/nginx/error.log`)


## Challenges & Troubleshooting

During the setup and testing phase, I ran into a few practical issues and resolved them:

* **502 Bad Gateway Error:** I encountered this error when testing Nginx while the background Python processes were stopped. Checking `/var/log/nginx/error.log` confirmed `Connection refused` to the upstream ports, which taught me how Nginx behaves when backend services are down.
* **Port Conflicts (`Address already in use`):** When restarting the Python servers, I ran into occupied ports. I used `sudo ss -lptn` and `pwdx` to track down the active PIDs, verify their working directories, and clean them up with `kill`.
* **Worker Processes & Round Robin Testing:** On a multi-core CPU, Nginx spawns multiple worker processes by default, which made quick sequential `curl` tests hit the same backend server repeatedly. I temporarily set `worker_processes 1;` in `nginx.conf` to observe strict 1-to-1 Round Robin alternation locally.





