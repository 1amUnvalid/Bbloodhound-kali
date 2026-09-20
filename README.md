# BloodHound CE — Kali Quick Fix Notes

## Issue 1: `bloodhound-start` hangs forever on "Please wait for the bloodhound service to start"

**Cause:** BloodHound API (`bhapi`) service is failing to start — usually a Neo4j auth mismatch.

**Check the real error:**
```bash
sudo systemctl status bloodhound
sudo journalctl -u bloodhound -n 50 --no-pager
```

If you see:
```
graph migration error: Neo4jError: Neo.ClientError.Security.Unauthorized
```
→ Neo4j's password doesn't match what's in `/etc/bhapi/bhapi.json` (`neo4j.secret` field, usually `test123`).

## Issue 2: Fixing Neo4j auth (reset to default)

Neo4j 4.4.x on Kali stores auth inside its internal `system` database (no `auth.ini` file to just delete). To reset:

```bash
sudo neo4j stop
sudo neo4j status   # confirm it's actually stopped

# Backup + remove the system DB (only resets auth, NOT your graph data)
sudo mv /etc/neo4j/data/databases/system /etc/neo4j/data/databases/system.bak
sudo mv /etc/neo4j/data/transactions/system /etc/neo4j/data/transactions/system.bak

sudo neo4j start
sleep 10
/usr/share/neo4j/bin/cypher-shell -u neo4j -p neo4j
```

It will force a password change on first login — set it to match `/etc/bhapi/bhapi.json`'s `neo4j.secret` value (default is `test123` unless changed).

Then restart BloodHound:
```bash
sudo systemctl restart bloodhound
sudo systemctl status bloodhound   # should show "active (running)" with no errors
curl -I http://localhost:8080      # 405 Method Not Allowed on HEAD = it's actually up, that's fine
```

Login at `http://localhost:8080` with `admin` / `admin` (forces password change).

## Issue 3: "WebGL Not Supported" on the Explore graph page (Chrome)

Common on VMs without GPU passthrough.

1. Go to `chrome://flags/#ignore-gpu-blocklist` → set to **Enabled**
2. Go to `chrome://flags/#enable-webgl` → set to **Enabled**
3. Click **Relaunch** at the bottom of the flags page
4. Reload `localhost:8080`

If still broken, run `chrome://gpu` to see the exact block reason.

## Useful diagnostic commands (in order)

```bash
sudo neo4j status                          # is Neo4j running
ps aux | grep neo4j                        # confirm process
curl -v http://localhost:7474              # Neo4j web interface reachable?
sudo docker ps                             # (not used — BloodHound CE on Kali is systemd, not Docker)
cat $(which bloodhound-start)              # see what the start script actually does
cat /etc/bhapi/bhapi.json                  # check expected DB/Neo4j creds
sudo systemctl status postgresql           # wrapper service, may show "exited" — normal
sudo pg_lsclusters                         # actual Postgres cluster status
sudo systemctl status bloodhound           # BloodHound API service status
sudo journalctl -u bloodhound -n 50 --no-pager
curl -I http://localhost:8080              # BloodHound web UI reachable?
```
