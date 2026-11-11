First lab is to understand what happens on the network when a machine connects to a network and make basic requests, very basic. (but i struggle to understand as if i never had a networking class before. i did though.)

| Device            | IP Address    |
| ----------------- | ------------- |
| **Kali Linux**    | `10.0.2.15`   |
| **Ubuntu Server** | `10.0.2.5`    |
| **Target Domain** | `example.com` |
command executed : `curl -I http://example.com`

The lab starts off with a simple ping to make sure both VMs is connected. This just consist of ICMP request and reply. 
### 1.**DHCP & ARP**
Before any web traffic could even happen, the machines had to figure out who was who on the local network:

- **DHCP:** Kali started off by asking for an IP address. The VirtualBox virtual router (`10.0.2.2`) responded with a DHCP ACK, leasing Kali the IP `10.0.2.15`. (At first I thought `10.0.2.2` was Ubuntu, but Ubuntu was actually `10.0.2.5`—`10.0.2.2` is VirtualBox's built-in router).
- **ARP:** Even though the machines know each other's IP addresses, physical network cards only talk using **MAC addresses**. Wireshark captured the classic shout:

    - _"Who has 10.0.2.15? Tell 10.0.2.2"_ (Broadcast to everyone)
    - _"10.0.2.15 is at 08:00:27:c3:fb:76"_ (Kali replying with its physical hardware address)
    - Once they know each other's MAC addresses, they save it to their ARP table so they don't have to keep asking.

![[Screenshot 2026-11-11 210845.png]]

### **2.DNS**
Before `curl` could touch `example.com`, it had to find out what IP address belonged to that name. Computers can't route packets to letters, only numbers.

Filtering by `dns` showed:

- Kali sending two questions at once:
    - **`A` record:** Asking for the IPv4 address.
    - **`AAAA` record:** Asking for the IPv6 address.
- The DNS server replied back with the IP: **`104.20.23.154`** (hosted on Cloudflare).
- Both the request and response shared the same Transaction ID (like a receipt number) so Kali knew which answer matched which question.
![[Pasted image 20261111211319.png]]

### **3.TCP 3-Way Handshake**

Now that Kali knew the IP address (`104.20.23.154`), it opened a reliable connection on standard HTTP port **80**.

Filtering by `tcp.port == 80` showed the classic 3-way high-five:

1. **`[SYN]`** — Kali →Server
2. **`[SYN, ACK]`** — Server →Kali
3. **`[ACK]`** — Kali →Server (connection established)

### **4.The HTTP Request & The ACK**

Once the line was open, the actual HTTP conversation took place:

- **`HEAD / HTTP/1.1`:** Kali asked for the headers. Because I used the `-I` flag in `curl`, it only asked for metadata, not the whole HTML website.
- **TCP `[ACK]` right after:** Interestingly, a plain TCP ACK packet appeared right after the request. This is TCP acting like a postal delivery confirmation—the server's network stack immediately confirmed _"I received your message"_ before the web server software even finished preparing the response.
- **`HTTP/1.1 200 OK`:** The server sent back the headers in plain text (status `200 OK`, date, server type).
![[Pasted image 20261111212349.png]]
### **5.Closing connection (`FIN, ACK`)**
Because `curl -I` only wanted the headers and nothing else, as soon as it got the `200 OK`, it was done.

Instead of keeping the port open, Kali initiated a clean teardown:

- Kali sent **`[FIN, ACK]`** (_"I'm finished, closing connection"_).
- The server acknowledged and closed its side with its own **`[FIN, ACK]`**.
- Connection cleanly close.
![[Pasted image 20261111212542.png]]