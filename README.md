Here is your **converted project-style README.md content in professional English**, based exactly on your structure, but rewritten as a proper project documentation.

You can directly copy-paste this into your `README.md`.

---

# 📘 DNS Server Configuration Project

## Authoritative DNS Setup Using BIND on Red Hat Enterprise Linux

---

# 🖥 Lab Setup

In this lab, I configured an **Authoritative DNS Server** using BIND and tested it with a client machine inside the same network.

## 🔹 Server Details

* IP Address: `192.168.1.38`
* Hostname: `node10.example.com`

## 🔹 Client Details

* IP Address: `192.168.1.85`
* Hostname: `node20.example.com`

---

# ✅ Server Side Configuration

---

## 1️⃣ Setting the Hostname (FQDN)

First, I configured the server’s Fully Qualified Domain Name:

```bash
hostnamectl set-hostname node10.example.com
exec bash
```

This ensures that the system hostname matches the DNS records that will be configured later.

---

## 2️⃣ Installing DNS (BIND)

I installed BIND and its utilities:

```bash
yum install bind bind-utils -y
```

Then I verified that the `named` user and group exist:

```bash
grep named /etc/passwd
grep named /etc/group
```

The DNS service runs under the `named` user, so its presence is required.

---

## 3️⃣ Starting and Enabling the DNS Service

```bash
systemctl start named
systemctl enable named
systemctl status named
```

* `start` → Starts the DNS service.
* `enable` → Ensures it starts automatically after reboot.
* `status` → Confirms that the service is running properly.

---

# 4️⃣ Editing the Main Configuration File

The primary DNS configuration file is:

```
/etc/named.conf
```

I edited it using:

```bash
vim /etc/named.conf
```

---

## 🔹 Important Changes in `/etc/named.conf`

```conf
options {
    listen-on port 53 { 192.168.1.38; };
    allow-query { any; };
    recursion no;
    dnssec-validation no;
};
```

---

## 🔎 Explanation of Options

### 🔹 listen-on port 53 { 192.168.1.38; };

* Specifies the IP address on which the DNS server will listen.
* Port 53 is the default DNS port.
* The server will accept DNS queries only on this IP.

---

### 🔹 allow-query { any; };

* Defines who can query this DNS server.
* `any` allows all clients to send DNS queries.

---

### 🔹 recursion no;

* `no` means this server acts as an **authoritative DNS server only**.
* It will not perform recursive lookups to resolve internet domains.

---

### 🔹 dnssec-validation no;

* Disables DNSSEC validation.
* Disabled here for lab simplicity.

---

# ❌ Commenting Default Zones

I commented out the default zones because I am manually defining my own zones:

```conf
//logging { };
//zone "." IN { };
//include "/etc/named.rfc1912.zones";
//include "/etc/named.root.key";
```

---

# 🔹 Copy Zone Templates (Using Default BIND Examples)

Instead of manually writing the forward and reverse zone structure from scratch, I referred to the default BIND zone examples available in:

```
/etc/named.rfc1912.zones
```

This file contains predefined sample zone configurations that help maintain correct syntax and structure.

---

## 📌 Step 1: Checking the Default Zone File

To review the default zone definitions, I used:

```bash
cat -n /etc/named.rfc1912.zones
```

This file contains example zone blocks such as:

* `localhost.localdomain`
* `1.0.0.127.in-addr.arpa`

These examples show the correct format for defining zones.

---

## 📌 Step 2: Extracting Only Required Template Lines

Instead of copying the entire file, I extracted only the relevant zone blocks:

```bash
sed -n '17,21p;35,39p' /etc/named.rfc1912.zones
```

This returned:

```conf
zone "localhost.localdomain" IN {
	type master;
	file "named.localhost";
	allow-update { none; };
};

zone "1.0.0.127.in-addr.arpa" IN {
	type master;
	file "named.loopback";
	allow-update { none; };
};
```

---

## 📌 Step 3: Using This as a Template

I used this structure as a reference and pasted it into:

```bash
vim /etc/named.conf
```

Then I modified it to match my own domain and network.

---

# 🔹 Why This Step Is Important

This step is important because:

* It ensures correct zone declaration syntax.
* It prevents configuration errors.
* It shows how BIND links a **zone name** to a **zone file path**.
* It helped me define:

```conf
zone "example.com" IN {
    type master;
    file "forward.zone";
    allow-update { none; };
};
```

and

```conf
zone "1.168.192.in-addr.arpa" IN {
    type master;
    file "reverse.zone";
    allow-update { none; };
};
```

Here:

* `file "forward.zone"` tells BIND to read records from `/var/named/forward.zone`
* `file "reverse.zone"` tells BIND to read records from `/var/named/reverse.zone`

So this template step helped me understand how forward and reverse zone files are connected to `named.conf`.

---

# 5️⃣ Adding Custom Zones

I added the following zone entries to `/etc/named.conf`:

```conf
zone "example.com" IN {
    type master;
    file "forward.zone";
    allow-update { none; };
};

zone "1.168.192.in-addr.arpa" IN {
    type master;
    file "reverse.zone";
    allow-update { none; };
};
```

---

## 🔎 Zone Explanation

### 🔹 Forward Zone – `example.com`

* Resolves **hostname → IP address**
* Zone file: `/var/named/forward.zone`

---

### 🔹 Reverse Zone – `1.168.192.in-addr.arpa`

* Resolves **IP address → hostname**
* Reverse format of network `192.168.1.0`
* Zone file: `/var/named/reverse.zone`

---

# 6️⃣ Creating Zone Files

To create the zone files, I copied existing templates:

```bash
cd /var/named
cp named.localhost forward.zone
cp named.loopback reverse.zone
chown named:named forward.zone
chown named:named reverse.zone
```

### Why This Step Is Important:

* Template files provide proper structure.
* Ownership must be set to `named:named`.
* Without correct permissions, DNS will fail to read zone files.

---

# 📘 Forward Zone Configuration

File:

```
/var/named/forward.zone
```

```conf
$TTL 1D
@   IN SOA  node10.example.com. root.example.com. (
        2026022201
        1D
        1H
        1W
        3H )

    IN NS  node10.example.com.

node10  IN A      192.168.1.38
node20  IN A      192.168.1.85
server  IN CNAME  node10
```

---

## 🔎 Important Notes

* The trailing dot (`.`) in FQDN is mandatory.
* Admin email in SOA:

  * `root@example.com`
  * Written as `root.example.com.` in zone file.

---

## Record Types Used

| Record | Purpose                   |
| ------ | --------------------------|
| SOA    | Start of Authority        |
| NS     | Name Server               |
| A      | Maps hostname to IPv4     |
| AAAA   | Maps hostname to IPv6     |
| CNAME  | Alias name                |

---

# 📘 Reverse Zone Configuration

File:

```
/var/named/reverse.zone
```

```conf
$TTL 1D
@   IN SOA  node10.example.com. root.example.com. (
        2026022201
        1D
        1H
        1W
        3H )

    IN NS  node10.example.com.

38  IN PTR node10.example.com.
85  IN PTR node20.example.com.
```

---

## 🔎 Reverse Zone Explanation

* Only the last octet of the IP is written.
* Example:

  * `38` → 192.168.1.38
  * `85` → 192.168.1.85
* PTR ( Pointer Record ) IP addresses to hostnames.

---

# 7️⃣ Checking Configuration

Before restarting, I validated the configuration:

```bash
named-checkconf
named-checkzone example.com /var/named/forward.zone
named-checkzone 1.168.192.in-addr.arpa /var/named/reverse.zone
```

This prevents syntax errors.

---

# 8️⃣ Restarting the DNS Service

```bash
systemctl restart named
systemctl status named
```

---

# 9️⃣ Firewall Configuration

To allow DNS traffic:

```bash
firewall-cmd --permanent --add-service=dns
firewall-cmd --reload
```

This opens port 53 for DNS queries.

---

# 🔟 Configuring Resolver (Server Side)

Edited:

```bash
vim /etc/resolv.conf
```

Added:

```conf
search example.com
nameserver 192.168.1.38
```

Now the server uses itself for DNS resolution.

---

# 🧪 Testing on Server

```bash
ping node20
nslookup node20
dig node20
host node20
```

---

# 🖥 Client Side Configuration

On the client machine:

```
IP: 192.168.1.85
Hostname: node20.example.com
```

---

## Configuring Resolver on Client

```bash
vim /etc/resolv.conf
```

Added:

```conf
search example.com
nameserver 192.168.1.38
```

The client now uses the DNS server for name resolution.

---

# 🧪 Testing on Client

```bash
ping node10
ping server
nslookup node10
dig node10
host node10
```

If successful, both forward and reverse lookups work correctly.

---

# 🎯 Final Summary

* Forward zone → Resolves hostname to IP.
* Reverse zone → Resolves IP to hostname.
* SOA and NS records define authority.
* Proper permissions on zone files are critical.
* Always validate configuration before restarting.
* DNS port 53 must be allowed in firewall.

---

# 🏁 Conclusion

In this project, I successfully configured an authoritative DNS server using BIND on Red Hat Enterprise Linux.

The setup now:

* Supports forward and reverse lookups.
* Allows internal clients to resolve hostnames.
* Simulates a real-world enterprise DNS environment.

This project strengthened my understanding of:

* DNS architecture
* Zone file structure
* BIND configuration
* DNS troubleshooting using `dig` and `nslookup`

---
