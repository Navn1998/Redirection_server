# High Availability Redirection Server with Linode IP Sharing

#OVERVIEW

When working with a customer who couldn't create an A record for Akamai CDN to access their domain due to internal DNS restrictions, I implemented a high-availability (HA) redirection server. This redirection server acts as a proxy between the customer and the Akamai CDN. To achieve HA, I leveraged Linode's IP sharing feature, and the server was built on CentOS. Below are the steps, commands, and HTML code to set this up.
The limitation this solved was the customer’s inability to create the required DNS CNAME/A record pointing their public hostname to Akamai’s edge hostname, due to internal DNS restrictions at the domain owner (BCCI), so a shared-IP redirection proxy was deployed to forward traffic to the Akamai CDN without needing that DNS change.

Why the DNS block mattered??
Standard Akamai onboarding expects a DNS CNAME from the site’s hostname to an Akamai edge hostname, which wasn’t possible here because the upstream DNS administrator would not create the record the integration required.​

Without permission to create or modify that DNS CNAME/A record at the authoritative zone, traffic could not be routed through Akamai directly, necessitating an alternate ingress under the team’s control.​

What the proxy did??
Two CentOS instances served a simple HTTP redirect that immediately sent users to the Akamai CDN URL, functioning as a proxy layer that the team could point users to without changing the BCCI-controlled DNS for the original hostname.​

The redirect page used a meta refresh and a fallback link to the CDN endpoint, enabling access flows to Akamai despite the lack of a first-party DNS CNAME to the edge hostname.
---

## Key Components

1. **Redirection Server**: Redirects incoming requests to the Akamai CDN.
2. **HA Configuration**: Uses Linode’s IP sharing feature to maintain high availability.
3. **CentOS**: The operating system running the redirection server.

---

## Steps to Build the Redirection Server

### 1. Set Up Your Linode Instances

- Create at least two Linode instances to enable high availability (e.g., `redirect-server-1` and `redirect-server-2`).
- Ensure both instances are in the same Linode data center.

### 2. Configure IP Sharing on Linode

- **IP sharing** allows multiple Linode instances to share a single IP address, ensuring continuity if one instance goes down.
- In your Linode Cloud Manager:
  1. Navigate to "Networking".
  2. Select the Linode instances you want to include in the IP sharing group.
  3. Assign the shared IP address to these instances.

For more details, refer to Linode's [IP Sharing Guide](https://www.linode.com/docs/guides/ip-failover-sharing/).

### 3. Install Necessary Software

```bash
# Update system packages
yum update -y

# Install Apache (used for HTTP redirection in this case)
yum install httpd -y

# Start and enable Apache
systemctl start httpd
systemctl enable httpd
```

### 4. Configure the Redirection

- Create a simple HTML page to handle the redirection. For example:

```html
<!DOCTYPE html>
<html>
<head>
    <meta http-equiv="refresh" content="0; URL='https://cdn.example.com'" />
    <title>Redirecting...</title>
</head>
<body>
    <p>If you are not redirected, <a href="https://cdn.example.com">click here</a>.</p>
</body>
</html>
```

- Save this file in the Apache web root directory (e.g., `/var/www/html/index.html`):

```bash
cat <<EOF > /var/www/html/index.html
<!DOCTYPE html>
<html>
<head>
    <meta http-equiv="refresh" content="0; URL='https://cdn.example.com'" />
    <title>Redirecting...</title>
</head>
<body>
    <p>If you are not redirected, <a href="https://cdn.example.com">click here</a>.</p>
</body>
</html>
EOF
```

- Adjust SELinux permissions if enabled:

```bash
chcon -R -t httpd_sys_content_t /var/www/html
```

### 5. Enable HA Using IP Sharing

- The Linode IP sharing feature ensures automatic failover by dynamically reassigning the shared IP to a healthy instance if the primary one goes down. No additional scripts or cron jobs are needed for this process. Both instances must be configured to use the floating IP and have identical redirection setups to maintain seamless service.

- This configuration allows the shared IP to remain functional as long as at least one instance is operational, ensuring high availability without manual intervention.

### 6. Testing

- Ensure the shared IP address is functional.
- Test failover by stopping the Apache service on one instance and confirming the other takes over seamlessly.

---



---

This solution leverages Linode’s IP sharing to achieve high availability while redirecting requests to Akamai CDN efficiently. The steps involved configuring SELinux, Firewalld, and creating an additional floating IP on the primary VM. IP sharing works such that if the primary VM with the floating IP shuts down, the IP is automatically reassigned to the secondary VM, ensuring continuity. Since Linode’s IP sharing manages this process, no CRON jobs were required for HA. Both VMs are configured to use this floating IP, and the redirection code is deployed on both servers for seamless failover.

