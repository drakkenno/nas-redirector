# Tunnel Verification Quick Reference

## What Was Created

Your NAS tunnel now has automatic verification and recovery capability via three new files:

### 1. **verify-tunnel.sh** - Main Verification Script
- Extracts NAS URL from `index.html`
- Tests if URL is accessible via HTTP
- Auto-runs `start-tunnel-and-update.sh` if tunnel is down
- Logs all activity to `/tmp/tunnel-verification.log`

### 2. **tunnel-verification.service** - Systemd Service
- Runs the verification script as a system service
- Can be enabled for automatic startup

### 3. **tunnel-verification.timer** - Systemd Timer
- Schedules verification to run every 5 minutes
- Integrates with the service above

---

## Usage

### Check Current Tunnel Status
```bash
cd /home/drakeno/git/nas-redirector
./verify-tunnel.sh --check
```

### Full Verification (Check + Auto-Recover)
```bash
./verify-tunnel.sh
```

### Force Tunnel Restart
```bash
./verify-tunnel.sh --recovery
```

### View Verification Logs
```bash
./verify-tunnel.sh --logs
# or
tail -f /tmp/tunnel-verification.log
```

### Setup Automatic Checks (Systemd)

```bash
# Copy service files to system
sudo cp /home/drakeno/git/nas-redirector/tunnel-verification.{service,timer} /etc/systemd/system/

# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable tunnel-verification.timer
sudo systemctl start tunnel-verification.timer

# View status
sudo systemctl status tunnel-verification.timer
sudo systemctl list-timers tunnel-verification.timer

# Monitor logs
sudo journalctl -u tunnel-verification.service -f
```

### Alternative: Setup Cron Job

Add to `crontab -e`:
```bash
# Check tunnel every 5 minutes
*/5 * * * * /home/drakeno/git/nas-redirector/verify-tunnel.sh >> /tmp/tunnel-cron.log 2>&1
```

---

## How It Works

### Verification Flow
```
1. Extract URL from index.html (const NAS_URL = "...")
2. Test if URL responds to HTTP request
3. Check for HTTP 2xx or 3xx response codes
4. Success? → Exit with status 0 ✅
5. Failure? → Try auto-recovery:
   a. Run start-tunnel-and-update.sh
   b. Extract new URL
   c. Test new tunnel
   d. Report result
```

### By Example

**Scenario 1: Tunnel is working**
```bash
$ ./verify-tunnel.sh --check
[2026-03-08 22:54:21] Testing URL: https://akpkl-79-117-251-118.a.free.pinggy.link
✅ URL is accessible (HTTP 200)

$ echo $?
0  # Success!
```

**Scenario 2: Tunnel is down, auto-recovery triggered**
```bash
$ ./verify-tunnel.sh
[2026-03-08 22:54:21] Testing URL: https://akpkl-79-117-251-118.a.free.pinggy.link
❌ URL returned HTTP 000
⚠️  NAS URL is not responding after 3 attempts
⚠️  Starting automatic tunnel recovery...
✅ Tunnel restart completed
✅ New tunnel URL: https://qzlfu-79-117-251-118.a.free.pinggy.link
✅ Tunnel is now operational
```

---

## Configuration

Edit `verify-tunnel.sh` to adjust timing:

```bash
MAX_RETRIES=3        # Attempts before declaring failure
RETRY_DELAY=5        # Seconds between retry attempts
# Adjust curl timeout in test_url() if needed:
--max-time 10        # Current: 10 second timeout per test
```

---

## Integration with Your Tunnel System

### Recommended Complete Setup

```bash
# 1. Start Pinggy with auto-refresh (handles 60-min expiration)
cd /home/drakeno/git/nas-redirector
./pinggy-auto-refresh.sh &

# 2. Enable periodic verification (handles unexpected failures)
sudo systemctl enable tunnel-verification.timer
sudo systemctl start tunnel-verification.timer

# Result:
# - Pinggy auto-refreshes every 50 minutes
# - Verification runs every 5 minutes
# - Any sudden failures detected & recovered within 5 minutes
# - Automatic fallback to Tailscale if Pinggy fails repeatedly
```

---

## Logs & Diagnostics

**Verification logs:**
```bash
tail -f /tmp/tunnel-verification.log
```

**Systemd logs:**
```bash
sudo journalctl -u tunnel-verification.service -f
```

**Check which tunnel is currently active:**
```bash
grep "const NAS_URL" /home/drakeno/git/nas-redirector/index.html
```

**Manual tunnel test:**
```bash
curl -I https://$(grep -oP 'const NAS_URL = "\K[^"]+' /home/drakeno/git/nas-redirector/index.html)
```

---

## Troubleshooting

**"URL returned HTTP 000"**
- Tunnel crashed or disconnected
- Auto-recovery will trigger
- Check if NAS server is running: `curl http://localhost:3000`

**Systemd timer not running**
```bash
# Check status
sudo systemctl status tunnel-verification.timer

# View timer schedule
sudo systemctl list-timers tunnel-verification.timer

# Start manually
sudo systemctl start tunnel-verification.service
```

**Auto-recovery keeps failing**
- Check `/tmp/tunnel-verification.log` for details
- Try manual recovery: `./verify-tunnel.sh --recovery`
- Check `start-tunnel-and-update.sh` works manually first

**Want to disable automatic checks temporarily**
```bash
sudo systemctl stop tunnel-verification.timer
sudo systemctl disable tunnel-verification.timer
```

---

## Files Reference

```
/home/drakeno/git/nas-redirector/
├── verify-tunnel.sh              ← Main verification script (EXECUTABLE)
├── tunnel-verification.service   ← Systemd service file
├── tunnel-verification.timer     ← Systemd timer file (5-minute interval)
├── TUNNEL_VERIFICATION.md        ← Full documentation
├── index.html                    ← Contains current NAS URL
└── start-tunnel-and-update.sh    ← Called by auto-recovery
```

---

## Summary

You now have **automated tunnel health monitoring** that:

✅ Checks tunnel status every 5 minutes (configurable)  
✅ Automatically recovers from failures  
✅ Integrates with systemd for reliability  
✅ Logs all activity for diagnostics  
✅ Works with Pinggy, Tailscale, or any URL  
✅ Can run via systemd timer or cron  

**Next step:** Consider setting up the systemd timer for hands-off operation!
