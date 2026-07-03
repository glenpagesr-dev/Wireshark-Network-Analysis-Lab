# 🦈 Wireshark & Network Analysis Lab

> **Lab 06 | Difficulty:** Beginner–Intermediate | **Estimated Time:** 2–4 Hours | **Tool:** Wireshark (Free) | **Certifications:** · Security+ · 

---

## 📺 [Watch Me Build the Lab Here](https://www.loom.com/share/914461e915e24080a18cb0c15f5ac811)

---

## 🎯 Objective

Networks carry every piece of data an organization produces — emails, database queries, login credentials, file transfers, API calls. When something goes wrong — a service is unreachable, a user reports slow performance, a security alert fires — the network is almost always involved. The only way to know what is actually happening on a network is to look at the packets. In this lab, I step into the role of a **Network/Security Analyst** using Wireshark to capture live traffic and answer the question: *what is actually happening on this wire?*

**By the end of this lab, you will have:**

- Captured live network traffic on a local interface
- Applied **display filters** to isolate DNS, TCP, and HTTP traffic in a busy capture
- Read a **TCP three-way handshake** (SYN → SYN-ACK → ACK) and recognized a failed connection attempt
- Identified **DNS queries and responses** and matched them against `nslookup` output
- Observed **cleartext credentials** in an HTTP POST request — hands-on proof of why HTTPS matters
- Reassembled a full conversation using **Follow TCP Stream**
- Saved and exported capture evidence for a GitHub portfolio repo

---

## 🏗️ Architecture Diagram

```
┌───────────────────────────────────────────────────────────────────────────┐
│                          YOUR NETWORK SEGMENT                             │
│                                                                           │
│   🌐 Internet / Remote Host (e.g. google.com, example.com)               │
│         │                                                                │
│         │  DNS Query  +  TCP/HTTP Traffic                                │
│         ▼                                                                │
│   ┌────────────────────────┐                                            │
│   │  📡 Router / Switch     │                                           │
│   └───────────┬─────────────┘                                           │
│               │ traffic reaches your NIC                                │
│               ▼                                                         │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │  💻 Local Machine (laptop / vm-wireshark-[yourname])              │ │
│   │                                                                   │ │
│   │   ┌────────────────────┐   Promiscuous Mode ON   ┌─────────────┐ │ │
│   │   │  🦈 Wireshark        │ ───────────────────────► │ Npcap /    │ │ │
│   │   │  Capture Engine     │                          │ libpcap    │ │ │
│   │   └──────────┬──────────┘                          └─────────────┘ │ │
│   │              │ raw packets, every frame                           │ │
│   │              ▼                                                    │ │
│   │   ┌───────────────────────────────────────────────────────────┐  │ │
│   │   │  🔍 Display Filter Engine                                  │  │ │
│   │   │  dns · tcp · http · ip.addr==x.x.x.x · tcp.flags.syn==1   │  │ │
│   │   └──────────┬────────────────────────────────────────────────┘  │ │
│   │              │                                                    │ │
│   │              ▼                                                    │ │
│   │   ┌───────────────┐   ┌────────────────┐   ┌────────────────────┐│ │
│   │   │ 📋 Packet List │   │ 📄 Packet Detail│   │ 🔢 Hex / Bytes Pane││ │
│   │   └───────────────┘   └────────────────┘   └────────────────────┘│ │
│   └───────────────────────────────────────────────────────────────────┘ │
│                                                                           │
│   ┌───────────────────────────────────────────────────────────────────┐ │
│   │  💾 Saved Evidence → dns-lookup.pcapng · tcp-handshake.pcapng ·   │ │
│   │     http-post-credentials.pcapng · tcp-stream-follow.pcapng      │ │
│   │     ────────────────────────────►  GitHub Portfolio Repo         │ │
│   └───────────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## ✅ Prerequisites

- [ ] A Windows, macOS, or Linux machine with an active network connection
- [ ] Local admin/sudo rights (required to install Npcap / capture packets)
- [ ] Ability to open a terminal (Command Prompt, Terminal, or shell)
- [ ] A test HTTP login form or test site for Exercise C — **never capture credentials on a system you don't own or have permission to test**

---

## 📋 Lab Variables (Naming Convention)

| Variable | Value |
|---|---|
| Capture Interface | Your active adapter — `Wi-Fi` or `Ethernet` |
| DNS Test Domain | `google.com` |
| HTTP Test Site | `http://example.com` |
| Capture File — DNS | `dns-lookup.pcapng` |
| Capture File — Handshake | `tcp-handshake.pcapng` |
| Capture File — Credentials | `http-post-credentials.pcapng` |
| Capture File — Stream | `tcp-stream-follow.pcapng` |
| Key Filters Used | `dns`, `tcp`, `http`, `tcp.flags.syn==1`, `http.request.method==POST` |
| Portfolio Repo | `Wireshark-Network-Analysis-Lab` |

---

## 🚀 Step-by-Step Instructions

### Phase 1 — Install Wireshark

> **Goal:** Get Wireshark and its packet-capture driver installed and verified before touching live traffic.

1. Go to [wireshark.org/download.html](https://www.wireshark.org/download.html) and download the installer for your OS.
2. **Windows:** Accept all defaults. Install **Npcap** when prompted — this is required to capture packets.
3. **macOS:** Run the `.dmg` installer. If prompted about **ChmodBPF**, allow it — this grants Wireshark permission to access network interfaces.
4. **Linux:** `sudo apt install wireshark` (Ubuntu/Debian), then add yourself to the wireshark group:
   ```bash
   sudo usermod -aG wireshark $USER
   # log out and back in for this to take effect
   ```
5. Verify the install:
   ```bash
   wireshark --version
   ```
<img width="1344" height="840" alt="Wireshark" src="https://github.com/user-attachments/assets/23f8c644-8306-4f6e-b48f-3206c19fc55d" />



**Expected Result:**
> ✅ Wireshark launches and the welcome screen lists your available capture interfaces.

---

### Phase 2 — Your First Capture

> **Goal:** Generate and observe a raw, unfiltered capture so you understand why display filters are necessary.

1. Open Wireshark. On the welcome screen you'll see a list of interfaces with wavy lines showing live activity.
2. Double-click your active interface (the one with the most wave activity).
3. Wireshark starts capturing immediately.
4. Open a browser and visit any website.
5. After 30 seconds, click the red square **Stop** button.

**Expected Result:**
> ✅ You have a capture with hundreds or thousands of packets from 30 seconds of browsing. This volume is exactly why display filters exist.

<img width="1334" height="516" alt="Packet Capture" src="https://github.com/user-attachments/assets/274fc789-80fb-4e33-928b-65dfef42a3dc" />



---

### Phase 3 — Apply Essential Display Filters

> **Goal:** Learn the handful of display filters that cover the majority of real-world troubleshooting and analysis.

**Display filters** (applied after capture) let you look at the same capture through different lenses without losing anything. **Capture filters** (applied before capture) limit what gets recorded. Always prefer display filters for this lab.

| Filter | What It Shows | When To Use It |
|---|---|---|
| `dns` | All DNS queries and responses | Troubleshooting name resolution |
| `http` | Unencrypted HTTP traffic only | Finding cleartext data |
| `tcp` | All TCP traffic | Starting point for connectivity investigations |
| `tcp.flags.syn == 1` | TCP SYN packets only | Seeing which hosts are attempting connections |
| `tcp.flags.reset == 1` | TCP RST packets | Finding refused/forcibly closed connections |
| `icmp` | Ping / reachability traffic | Verifying basic connectivity |
| `ip.addr == 192.168.1.1` | Traffic to/from a specific IP | Isolating one host in a busy capture |
| `tcp.port == 443` | HTTPS traffic | Identifying encrypted web traffic |
| `http.request` | HTTP GET/POST requests only | Finding web requests, spotting exfiltration |

---

### Phase 4 — Exercise A: Capture a DNS Lookup

> **Goal:** Capture a live DNS query/response pair and confirm the resolved IP matches a terminal lookup.

1. Start a capture on your active interface.
2. Open a **separate terminal window** (Wireshark has no terminal built in) and run:
   ```bash
   nslookup google.com
   ```
3. Switch back to Wireshark and click **Stop**.
4. Apply the filter `dns` and press Enter.
5. Find the query packet — Info column shows `Standard query A google.com`.
6. Find the response packet — Info column shows `Standard query response A google.com`.
7. Click the response packet, expand **Domain Name System (response)** in the detail pane, then expand **Answers** to see the A record IP address. Confirm it matches your terminal output.

<img width="1342" height="508" alt="DNS lookup" src="https://github.com/user-attachments/assets/0dcae038-cab1-4e16-86e3-dccf030c174e" />

<img width="1306" height="514" alt="DNS Lookup 2" src="https://github.com/user-attachments/assets/f67c62ca-9997-413b-82bd-7c9fb7507368" />


**Expected Result:**
> ✅ The IP address in the Wireshark Answers section matches the IP returned by `nslookup` in your terminal.



*Best screenshot moment: right after step 7, with the Answers section expanded and the matching IP visible next to your terminal output.*

---

### Phase 5 — Exercise B: Watch the TCP Three-Way Handshake

> **Goal:** Identify the SYN → SYN-ACK → ACK sequence that establishes every reliable TCP connection.

1. Start a capture.
2. Run `nslookup example.com` in your terminal to get the IP address.
3. Open a browser and navigate to `http://example.com` (HTTP, not HTTPS — easier to see the handshake).
4. Stop the capture.
5. Apply the filter: `tcp and ip.addr == [that IP]`
6. Find the three sequential packets in order:

| Packet | Flags | Meaning |
|---|---|---|
| 1st | `SYN` | Your machine: "I want to connect." |
| 2nd | `SYN, ACK` | Server: "Got it. Connection accepted." |
| 3rd | `ACK` | Your machine: "Confirmed. Ready to send data." |

> A SYN with no SYN-ACK means the connection was refused or the server is unreachable. A RST packet means the connection was forcibly closed. These are the two most common patterns network engineers look for.

<img width="1299" height="492" alt="TCP Handshake1" src="https://github.com/user-attachments/assets/2a2575d6-b51b-4a40-b075-dc1eb057a009" />

<img width="540" height="142" alt="TCP Handshake 2" src="https://github.com/user-attachments/assets/6708bb3d-c9a4-40e6-9ba7-3f79ee9bfa8c" />



**Expected Result:**
> ✅ All three handshake packets are visible in sequence, confirming a successful TCP connection before any HTTP data flows.

![TCP three-way handshake](screenshots/tcp-handshake.png)

*Best screenshot moment: shift-click all three packets so SYN, SYN-ACK, and ACK are visible together in one shot.*

---

### Phase 6 — Exercise C: Spot Cleartext Credentials (HTTP)

> **Goal:** See firsthand why unencrypted HTTP login forms expose credentials to anyone on the network path.

> ⚠️ **Important:** Educational exercise only. Only capture on networks and systems you own or have explicit permission to test. Never run this against systems you don't own.

1. Set up a test HTTP login form on your local machine, or use a test site running over HTTP (not HTTPS).
2. Start a capture.
3. Submit the login form with a **test** username and password only.
4. Stop the capture.
5. Apply the filter: `http.request.method == POST`
6. Click the POST packet and expand **HTML Form URL Encoded** in the detail pane.
7. You'll see the username and password in plaintext.

<img width="1304" height="786" alt="Cleartext Credentials" src="https://github.com/user-attachments/assets/51b226a6-0b7d-445a-96c9-9b6ba06d2e4a" />




**Expected Result:**
> ✅ The username and password submitted in step 3 appear in cleartext inside the HTML Form URL Encoded section — proof the data was never encrypted in transit.

![HTTP POST cleartext credentials - test account only](screenshots/http-post-credentials.png)

*Best screenshot moment: HTML Form URL Encoded expanded. **Blur or redact the credential values before publishing** if there is any chance they resemble a real account.*

This is exactly why every login form must use HTTPS. Without TLS, anyone on the network path — your ISP, a coffee shop router, or an attacker performing a man-in-the-middle — can read credentials exactly as typed.

---

### Phase 7 — Exercise D: Follow a Full TCP Stream

> **Goal:** Reassemble an entire client-server conversation into one readable view instead of dozens of individual packets.

1. Capture any HTTP traffic by navigating to an HTTP website.
2. Find any HTTP packet in the capture list.
3. Right-click it → **Follow** → **TCP Stream**.
4. Wireshark reassembles all packets from that connection into a readable conversation. Red text is your browser's request; blue text is the server's response.

**Expected Result:**
> ✅ The full request/response conversation is readable top to bottom in a single window, with request and response color-coded.

<img width="1280" height="770" alt="Follow TCP Stream" src="https://github.com/user-attachments/assets/3c9ace8b-5aed-4390-a3fc-b4b0f392678c" />


![Follow TCP Stream conversation](screenshots/tcp-stream-follow.png)

*Best screenshot moment: the Follow TCP Stream window, fully scrolled to show both the red request and blue response.*

---

### Phase 8 — Save and Export Captures

> **Goal:** Turn your live captures into portable evidence files ready for your GitHub portfolio.

```bash
# Save a capture for later analysis
File → Save As → choose .pcapng format

# Export only the packets matching your current filter
Apply your display filter first
File → Export Specified Packets → Displayed

# Re-open a saved capture
File → Open → select your .pcapng file

# Command-line capture with tshark (included with Wireshark)
tshark -i eth0 -w capture.pcapng -c 1000
# -i: interface   -w: output file   -c: stop after this many packets
```

**Expected Result:**
> ✅ Four `.pcapng` evidence files exist on disk: `dns-lookup.pcapng`, `tcp-handshake.pcapng`, `http-post-credentials.pcapng`, and `tcp-stream-follow.pcapng`.

![Four saved evidence captures ready for upload](screenshots/saved-captures-folder.png)

*Best screenshot moment: your file browser showing all four saved `.pcapng` files, right before uploading to GitHub.*

---

## 🔧 Troubleshooting

| Issue | Root Cause | Fix |
|---|---|---|
| No interfaces show wave activity | Npcap not installed (Windows) or missing permissions | Reinstall Wireshark and select "Install Npcap"; on macOS/Linux confirm ChmodBPF/group permissions |
| "You don't have permission to capture on this device" | User not in the `wireshark` group (Linux) or not elevated (Windows/macOS) | Run `sudo usermod -aG wireshark $USER`, log out/in; on Windows launch Wireshark as Administrator once |
| Only seeing my own traffic, no other hosts | Modern switched networks only forward traffic addressed to you — this is expected | Use `ip.addr ==` filters to isolate hosts you have visibility into, or set up port mirroring/SPAN for broader visibility |
| `dns` filter shows nothing after `nslookup` | Capture wasn't running when `nslookup` was executed, or wrong interface selected | Start the capture on the active (wavy-line) interface **before** running the command |
| Can't find the POST packet in Exercise C | Filter typo, or the test form used GET instead of POST | Re-check the filter is exactly `http.request.method == POST`; confirm the form's method is POST |

---

## 🧹 Clean Up & Secure Your Evidence

> ⚠️ **Important:** Never commit real credentials, production traffic, or third-party company data to a public GitHub repo. Exercise C must only ever contain test credentials, and screenshots of it should be reviewed for redaction before publishing.

1. Stop and close any live captures in Wireshark.
2. Confirm your four saved files: `dns-lookup.pcapng`, `tcp-handshake.pcapng`, `http-post-credentials.pcapng`, `tcp-stream-follow.pcapng`.
3. Create your `Wireshark-Network-Analysis-Lab` GitHub repo, add a `screenshots/` folder, and upload this README along with your capture files and screenshots.
4. Add the Loom video link to the top of this README once recorded.

---

## 💡 Key Concepts Covered

| Concept | What It Does |
|---|---|
| **Packet** | The small unit of data — header (source/destination/port) + payload — that Wireshark captures and displays individually |
| **Protocol** | A set of rules for formatting and transmitting data (DNS, HTTP, TCP, ICMP each handle a different job) |
| **TCP Three-Way Handshake** | The SYN → SYN-ACK → ACK sequence that opens a reliable TCP connection before data flows |
| **DNS (Domain Name System)** | Translates domain names into IP addresses; happens before nearly every network action |
| **HTTP vs HTTPS** | HTTP is unencrypted and readable by anyone on the path; HTTPS adds TLS encryption to protect the payload |
| **Promiscuous Mode** | The NIC mode Wireshark enables to capture all traffic on the segment, not just traffic addressed to you |
| **Display Filters vs Capture Filters** | Display filters narrow what you *see* after capture (non-destructive); capture filters limit what gets *recorded* |

---

## 🛠️ Tools & Services Used

![Wireshark](https://img.shields.io/badge/Wireshark-1679A7?style=for-the-badge&logo=wireshark&logoColor=white)
![Npcap](https://img.shields.io/badge/Npcap-0078D4?style=for-the-badge&logo=windows&logoColor=white)
![TCP/IP](https://img.shields.io/badge/TCP%2FIP-333333?style=for-the-badge&logo=cisco&logoColor=white)
![DNS](https://img.shields.io/badge/DNS-4A90D9?style=for-the-badge&logo=internetcomputer&logoColor=white)
![HTTP/HTTPS](https://img.shields.io/badge/HTTP%2FHTTPS-25A162?style=for-the-badge&logo=letsencrypt&logoColor=white)

---

## 👤 Author

**Glen Page** — Cloud Engineer

📎 [GitHub](https://github.com/glenpagesr-dev) | 🔗 [LinkedIn](https://linkedin.com/in/glen-page-862730246)

---

*Part of my Cloud Engineering portfolio. Built to demonstrate hands-on packet analysis skills aligned to Network+ and  Security+ *
