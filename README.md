Enterprise Production-Hardened Nginx Load Balancer Cluster

A high-availability, multi-node web infrastructure engineered on CentOS Stream 10. This project demonstrates a production-grade enterprise design pattern: isolating application backends within a private subnet and deploying a secure, resilient public-facing edge gateway to handle traffic routing, cryptographic offloading, and denial-of-service mitigation.

🏗️ Architectural Topology

```
                 [ Physical Host Laptop / Web Browser ]

                                  │

                     (Public/Bridged Net: Port 443)

                                  ▼

                    ┌──────────────────────────────┐

                    │     Nginx Load Balancer      │  (nginx-lb)

                    │ 192.168.56.103 (Bridged IP)  │

                    └──────────────┬───────────────┘

                                   │

                       (Host-Only Net: Port 80)

                                   ▼

        ┌──────────────────────────┴──────────────────────────┐

        ▼                                                     ▼

┌──────────────────────────────┐                      ┌──────────────────────────────┐

│     Backend Web Server 01    │                      │     Backend Web Server 02    │

│  192.168.56.101 (Private)    │                      │  192.168.56.102 (Private)    │

└──────────────────────────────┘                      └──────────────────────────────┘
```

🛠️ Production-Ready Features

1. Layer 7 Load Balancing (Round-Robin)

   The edge gateway uses an upstream routing pool to distribute incoming application workloads sequentially across the cluster. This architecture ensures horizontal scalability and fault tolerance—if a backend node drops offline, the gateway transparently reroutes traffic without user disruption.

2. SSL/TLS Termination Proxy

* Cryptographic Offloading: The load balancer intercepts encrypted traffic over Port 443, handles the CPU-intensive cryptographic handshake using a 2048-bit RSA key suite, and passes unencrypted HTTP packets over the isolated private switch. This preserves backend node compute resources.

* HSTS-Style Forced Upgrades: Implements an automated HTTP 301 Permanent Redirect that catches unencrypted requests on Port 80 and upgrades them structurally to safe ```https://``` protocols.

* Cipher Hardening: Explicitly restricts connections to modern, secure protocols (TLSv1.2 and TLSv1.3) while disabling legacy, vulnerable ciphers.

3. Edge Defense \& Throttling

* Algorithmic Rate Limiting: Utilizes a dedicated 10MB shared memory tracking zone (```limit\_req\_zone```) pinned to the client's binary remote IP address. Traffic is restricted to a maximum threshold of 5 requests per second with a managed burst buffer. Spammed or automated script attacks are instantly blunted with an HTTP 429 Too Many Requests defense layer.

* Information Disclosure Protection: Mitigates banner-grabbing exploits by disabling ```server\_tokens```. The reverse proxy completely redacts internal software versions from error screens and response headers.

* Clickjacking \& MIME Protection: Injects hardening security headers (```X-Frame-Options: DENY, X-Content-Type-Options: nosniff```) to block malicious framing and script execution vectors.

🛡️ Enterprise Engineering Challenges Overcome

An infrastructure deployment is only as stable as the systems engineering underlying it. During the provisioning of this RHEL-family cluster, two critical system blockades were structurally resolved:

🔀 Challenge 1: The Network Interface Provisioning Hurdle

* Symptom: The load balancer was completely unable to forward proxy requests to the backend nodes, resulting in catastrophic ```110: Connection timed out``` logs.

* Root Cause: CentOS Stream 10 initialized the secondary Host-Only virtual interface network adapter (```enp0s8```) in an unmanaged, unconfigured state on boot, depriving the machine of its private cluster identity.

* Resolution: Leveraged ```nmcli``` (NetworkManager CLI) to structurally inject a permanent, automated system connection profile mapped explicitly to the underlying physical device hardware kernel state:

```
sudo nmcli connection add type ethernet ifname enp0s8 con-name "Host-Only"
sudo nmcli connection modify "Host-Only" connection.autoconnect yes
sudo nmcli connection up "Host-Only"
```

🔒 Challenge 2: The SELinux Reverse Proxy Blockade

* Symptom: Even with correct Nginx reverse proxy syntax and fully verified network pathways, the load balancer triggered an absolute ```Permission Denied``` drop inside ```/var/log/nginx/error.log``` when attempting to route traffic.

* Root Cause: Security-Enhanced Linux (SELinux) enforces a strict default-deny mandatory access control architecture. It identified the unprivileged ```nginx``` worker daemon attempting to initiate outbound network sockets to foreign private IPs and immediately neutralized the connection.

* Resolution: Avoided the insecure shortcut of disabling SELinux. Instead, flipped an administrative binary policy flag persistently to permanently teach the kernel that the proxy relay behavior is authorized:

```sudo setsebool -P httpd\_can\_network\_connect 1```

🚦 Verification \& Stress Testing

Infrastructure integrity was validated by generating high-concurrency loops from an adjacent node on the private subnet to simulate a brute-force layer 7 spam attack:

```for i in {1..20}; do curl -k -I \[https://192.168.56.103](https://192.168.56.103); done```

Verification Logs Result:

```
HTTP/1.1 200 OK (Traffic distributed cleanly to backend nodes)
HTTP/1.1 200 OK
HTTP/1.1 200 OK
HTTP/1.1 200 OK
HTTP/1.1 429 Too Many Requests (Algorithmic rate limiter aggressively defending the edge)
HTTP/1.1 429 Too Many Requests
```
