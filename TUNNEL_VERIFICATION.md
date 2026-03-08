# NAS Tunnel Verification & Auto-Recovery

Automated verification script that checks if your NAS tunnel is working and automatically restarts it if needed.

## Features

✅ **Automatic URL Extraction** - Reads the configured URL directly from `index.html`  
✅ **Health Checks** - Tests HTTP accessibility of your tunnel  
✅ **Auto-Recovery** - Automatically runs `start-tunnel-and-update.sh` if tunnel is down  
✅ **Retry Logic** - Multiple attempts before declaring failure  
✅ **Logging** - Full activity logged to `/tmp/tunnel-verification.log`  
✅ **Systemd Integration** - Optional periodic checks via timer  

## Quick Start

### Manual Verification

```bash
# Check if tunnel is working (no recovery)
./verify-tunnel.sh --check

# Full verification with auto-recovery if needed
./verify-tunnel.sh

# Force restart tunnel
./verify-tunnel.sh --recovery

# View verification logs
./verify-tunnel.sh --logs
```

### Automatic Periodic Checks (Systemd Timer)

Setup automatic verification every 5 minutes:

```bash
# Copy service files
sudo cp tunnel-verification.service /etc/systemd/system/
sudo cp tunnel-verification.timer /etc/systemd/system/

# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable tunnel-verification.timer
sudo systemctl start tunnel-verification.timer

# Check status
sudo systemctl status tunnel-verification.timer
sudo systemctl list-timers tunnel-verification.timer

# View logs
sudo journalctl -u tunnel-verification.service -f
```

### As a Cron Job

Alternative to systemd timer - add to crontab for periodic checks:

```bash
# Edit crontab
crontab -e

# Add this line to check every 5 minutes
*/5 * * * * /home/drakeno/git/nas-redirector/verify-tunnel.sh >> /tmp/tunnel-verification-cron.log 2>&1
```

## Configuration

Edit `verify-tunnel.sh` to customize:

```bash
MAX_RETRIES=3          # Number of attempts before recovery
RETRY_DELAY=5          # Seconds between retries
LOG_FILE=/tmp/tunnel-verification.log  # Where to store logs
```

## Behavior

### Check Only Mode (`--check`)
1. Extracts URL from `index.html`
2. Tests if URL responds with HTTP 2xx or 3xx
3. Returns 0 (success) or 1 (failure)
4. No automatic recovery

### Full Verification (default)
1. Extracts URL from `index.html`
2. Tests accessibility (max 3 attempts, 5s between retries)
3. If fails: Runs `start-tunnel-and-update.sh` to establish new tunnel
4. Tests new tunnel
5. Logs results

## Log Examples

**Successful tunnel:**
```
✅ URL is accessible (HTTP 200)
✅ Tunnel is healthy and operational
```

**Failed tunnel, auto-recovered:**
```
❌ URL returned HTTP 000
⚠️  NAS URL is not responding after 3 attempts
⚠️  Starting automatic tunnel recovery...
✅ Tunnel restart completed
✅ New tunnel URL: https://qzlfu-79-117-251-118.a.free.pinggy.link
✅ Tunnel is now operational
```

## Understanding HTTP Codes

- **200-399**: Tunnel is working ✅
- **000**: Connection timeout/refused - tunnel likely crashed 🔄
- **5xx**: Server error - NAS app may have crashed
- **4xx**: Client error - check URL format

## Troubleshooting

**Script says tunnel failed, but it's actually working:**
- Pinggy tunnels take a few seconds to stabilize after creation
- Increase `RETRY_DELAY` in the script
- Increase `--max-time` in curl command

**Recovery keeps failing:**
- Check `/tmp/tunnel-verification.log` for details
- Check `start-tunnel-and-update.sh` runs manually
- Verify NAS server is running: `curl localhost:3000`

**Systemd timer not running:**
```bash
# Check timer status
sudo systemctl status tunnel-verification.timer

# Check if service runs manually
sudo systemctl start tunnel-verification.service
sudo journalctl -u tunnel-verification.service -n 50
```

## Files

- `verify-tunnel.sh` - Main verification script
- `tunnel-verification.service` - Systemd service unit
- `tunnel-verification.timer` - Systemd timer (runs every 5 min)
- `/tmp/tunnel-verification.log` - Verification logs

## Integration with Other Scripts

The verification script works with:
- `start-tunnel-and-update.sh` - Auto-recovery mechanism
- `index.html` - Reads URL configuration
- `pinggy-auto-refresh.sh` - Can be paired for 60-min Pinggy expiration

## Recommended Setup

For production NAS access:

```bash
# 1. Setup auto-refresh for Pinggy (60-min expiration)
./pinggy-auto-refresh.sh &

# 2. Setup periodic verification (5-min checks)
sudo systemctl enable tunnel-verification.timer
sudo systemctl start tunnel-verification.timer

# 3. This ensures:
# - Pinggy URL refreshed before expiration
# - Any tunnel failures detected and recovered within 5 minutes
# - Automatic fallback to Tailscale if Pinggy fails
```
