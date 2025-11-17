# Installing Neko on TrueNAS SCALE via Custom App

> **About Neko:** [Neko](https://github.com/m1k1o/neko) is a self-hosted virtual browser that runs in Docker and uses WebRTC. Created by [m1k1o](https://github.com/m1k1o).

This guide walks through installing Neko on TrueNAS SCALE using the Custom App feature. Perfect for homelabbers who want to run Neko alongside other services on their NAS.

## Prerequisites

- TrueNAS SCALE (Community Edition) installed and accessible
- A storage pool configured
- Access to TrueNAS web interface
- Basic knowledge of your network IP addresses

## Installation Steps

### 1. Access Apps Section
1. Log into your TrueNAS web interface
2. Navigate to **Apps** in the left sidebar
3. If first time using Apps, select your storage pool when prompted

### 2. Create Custom App
1. Click **Discover Apps**
2. Click **Custom App** button

### 3. Configure Basic Settings

**Application Name:** `neko`

**Image Repository:** `m1k1o/neko`

**Image Tag:** `firefox`

> **Note:** You can replace `firefox` with `chromium`, `google-chrome`, `brave`, or `edge` depending on your browser preference.

### 4. Configure Networking

**Option A: Host Network Mode (Recommended)**
1. In the **Networking** section, find **Network Mode** or **Host Networking**
2. Enable **Host Network** mode
3. This allows the container to use the host's network directly

**Option B: Default Networking with Port Mappings**
If Host Network mode is not available or doesn't work:
1. Leave networking on default settings
2. Add the following port mappings:
   - **Container Port:** `8080` → **Host Port:** `8080` (Protocol: TCP)
   - **Container Port:** `52000` → **Host Port:** `52000` (Protocol: UDP)

### 5. Configure Environment Variables

Add the following environment variables (click **Add** for each):

| Variable Name | Value | Description |
|--------------|--------|-------------|
| `NEKO_SCREEN` | `1920x1080@30` | Screen resolution and framerate |
| `NEKO_PASSWORD` | `your_user_password` | Regular user password |
| `NEKO_PASSWORD_ADMIN` | `your_admin_password` | Admin password (must be different from user) |
| `NEKO_EPR` | `52000-52100` (Host Network) or `52000` (Port Mapping) | UDP port range for WebRTC |
| `NEKO_NAT1TO1` | `192.168.x.x` | Your TrueNAS server's local IP |

> **Important:** The admin and user passwords **must be different** to avoid authentication issues.

### 6. Deploy the Application
1. Scroll to bottom and click **Save** or **Install**
2. Wait for deployment to complete (this may take 5-20 minutes as the Docker image downloads)
3. Status will show green "Running" when ready

### 7. Access Neko
1. Open your web browser
2. Navigate to: `http://your-truenas-ip:8080`
3. Enter your username and password
4. Click the **hand/control icon** to take control of the browser

## Exposing Neko Externally (Optional)

### Using Nginx Proxy Manager (NPM)

If you want to access Neko via a domain name using NPM:

**NPM Proxy Host Configuration:**
- **Domain Name:** `neko.yourdomain.com`
- **Scheme:** `http`
- **Forward Hostname/IP:** `your-truenas-local-ip`
- **Forward Port:** `8080`
- **✅ Enable Websockets Support** (Critical!)
- Enable SSL certificate

**Router Port Forwarding:**
- Forward ports `80` and `443` (TCP) to your NPM server

**Optional - UDP Port Forwarding:**
Some documentation mentions forwarding UDP ports `52000-52100` to your TrueNAS IP for WebRTC connections. However, in testing, Neko works perfectly fine externally (including video and audio) through NPM without this additional forwarding. NPM appears to handle the WebRTC connections through the standard HTTPS port. You can try without it first, and only add UDP forwarding if you experience connection issues.

**Update Environment Variable for External Access:**
After setting up external access, edit your Neko app and change:
- `NEKO_NAT1TO1` to your **public IP address** or **DDNS hostname**

This ensures WebRTC connections work properly from outside your network.

## Troubleshooting

**App stuck at 60% during installation:**
- This is normal - the Docker image is large (1-3GB)
- Wait 15-20 minutes for download to complete
- Don't refresh or cancel

**Cannot click or control the browser:**
- Look for the **hand/control icon** in the Neko interface and click it
- Log in with your admin password for automatic control
- Ensure browser zoom is at 100%

**Authentication issues:**
- Verify admin and user passwords are different
- Try logging in with admin credentials

**Connection issues when accessing externally:**
- Verify WebSockets are enabled in NPM
- Check `NEKO_NAT1TO1` is set to your public IP or DDNS hostname
- Ensure SSL certificate is properly configured
- If video/audio still doesn't work, try forwarding UDP ports 52000-52100 as mentioned above

## Browser Options

Change the `Image Tag` to use different browsers:
- `firefox` - Most stable, recommended
- `chromium` - Lightweight option
- `google-chrome` - Full Chrome experience
- `brave` - Privacy-focused
- `edge` - Microsoft Edge

## Performance Tips

- Adjust `NEKO_SCREEN` to lower resolution for better performance: `1280x720@30`
- Use Firefox image for best stability
- Ensure adequate resources allocated to TrueNAS Apps

## Additional Resources

- [Neko Documentation](https://neko.m1k1o.net/)
- [Neko GitHub](https://github.com/m1k1o/neko)
- [TrueNAS SCALE Documentation](https://www.truenas.com/docs/scale/)

---

Successfully running Neko on TrueNAS SCALE? Feel free to share your setup or ask questions below!
