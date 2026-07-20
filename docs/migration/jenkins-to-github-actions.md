# Migrate `task.management` CI/CD from Windows Jenkins to Self-Hosted GitHub Actions on Ubuntu VPS (Multi-Site Ready)

Replace the Windows/Jenkins/IIS deployment pipeline for `Api.Server` + `api.client` with a self-hosted GitHub Actions runner on the Ubuntu VPS that builds, publishes, and zero/minimal-downtime-deploys the app behind nginx, while establishing a reusable convention so additional sites can be added later without reworking this setup.

**Confirmed specifics:**
- Repo: `https://github.com/Devnity-Rafal-Labecki/task.management` (branch `master`)
- Systemd service: `taskmanagement.service`
- Domain: `localtasklist.com` — **DNS not migrated yet**; site is currently reached only via the server's bare IP, over HTTP (no SSL yet)
- Current nginx: `default_server` on port 80 catches all requests to the IP (no `server_name` matching needed)
- .NET: 10 — **only the runtime was pre-installed on the VPS, not the SDK**; `dotnet restore/build/publish` in Section 6's workflow require the SDK (see Section 5b)
- Kestrel port: `5000` (nginx proxies → `127.0.0.1:5000`)
- App root: `/var/www/taskmanagement`
- App/CI user: `githubrunner` (already created)
- Deployment strategy: **zero/minimal downtime** via release-folder + symlink swap
- **Multi-site**: no second site exists yet, but the plan defines a port/naming convention so future apps can be added cleanly (see Section 8)
- **Runner scope**: org-level (`Devnity-Rafal-Labecki`), label `linux-vps` — one shared runner for all repos, see Section 5

> **SSL follow-up**: Let's Encrypt/certbot setup is deferred until `localtasklist.com` DNS actually points at this VPS (see Section 9). Not part of this CI/CD migration.

---

## 1. Inspect current server state (read-only, run as yourself or `githubrunner`)
**Run as:** your sudo-capable admin user (or `githubrunner` — read-only, no changes made).
```bash
systemctl status taskmanagement.service
sudo cat /etc/systemd/system/taskmanagement.service
sudo ls -la /var/www/taskmanagement
sudo ls /etc/nginx/sites-enabled/
sudo cat /etc/nginx/sites-enabled/*taskmanagement* 2>/dev/null || sudo cat /etc/nginx/sites-enabled/default
dotnet --list-runtimes
id githubrunner
```
Confirm: `WorkingDirectory`/`ExecStart` path in the unit file, current owner/permissions of `/var/www/taskmanagement`, that nginx `proxy_pass` targets `http://127.0.0.1:5000`, and which file currently holds the `default_server` directive (needed for Section 6).

## 2. Restructure app folder for symlink-based releases
**Run as:** sudo/root — one-time manual setup, done directly on the VPS (not by `githubrunner`).
Move to a `releases/<n>` + `current` symlink layout so deploys don't overwrite a running app in place.
```bash
sudo systemctl stop taskmanagement.service
sudo mkdir -p /tmp/legacy
sudo mv /var/www/taskmanagement/* /tmp/legacy/
sudo mkdir -p /var/www/taskmanagement/releases
sudo mv /tmp/legacy /var/www/taskmanagement/releases/legacy
sudo ln -s /var/www/taskmanagement/releases/legacy /var/www/taskmanagement/current
```
Update the systemd unit's `WorkingDirectory` and `ExecStart` to point at `/var/www/taskmanagement/current`:
```bash
sudo sed -i 's|^WorkingDirectory=.*|WorkingDirectory=/var/www/taskmanagement/current|' /etc/systemd/system/taskmanagement.service
sudo sed -i 's|^ExecStart=/usr/bin/dotnet .*|ExecStart=/usr/bin/dotnet /var/www/taskmanagement/current/Api.Server.dll|' /etc/systemd/system/taskmanagement.service
sudo cat /etc/systemd/system/taskmanagement.service  # verify both lines changed correctly
```
If the `ExecStart` line doesn't match that pattern (different binary path/args), edit it manually instead: `sudo nano /etc/systemd/system/taskmanagement.service`. Then:
```bash
sudo systemctl daemon-reload
sudo systemctl start taskmanagement.service
sudo systemctl status taskmanagement.service
```
Verify site still loads at `http://YOUR_VPS_IP/` (no domain/SSL yet).

## 3. Rename nginx site file to the per-app convention (multi-site prep)
**Run as:** sudo/root — one-time manual setup.
Rename/normalize the existing config so future sites follow the same pattern instead of relying on an anonymous `default` file:
```bash
sudo mv /etc/nginx/sites-available/default /etc/nginx/sites-available/taskmanagement
# (skip the mv if it's already named appropriately; just ensure the filename == app name)
sudo rm -f /etc/nginx/sites-enabled/default
sudo ln -s /etc/nginx/sites-available/taskmanagement /etc/nginx/sites-enabled/taskmanagement
```
Rewrite `/etc/nginx/sites-available/taskmanagement` — keep `listen 80 default_server;` for now (it's the only site and must stay reachable by bare IP), but add a `server_name` line ready for cutover once DNS resolves. Write it non-interactively with a heredoc (back up the original first):
```bash
sudo cp /etc/nginx/sites-available/taskmanagement /etc/nginx/sites-available/taskmanagement.bak
sudo tee /etc/nginx/sites-available/taskmanagement > /dev/null <<'EOF'
server {
    listen 80 default_server;
    server_name localtasklist.com www.localtasklist.com _;

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
EOF
```
The trailing `_` catches the bare-IP/unmatched-Host case, so this keeps working exactly as today. Test and reload:
```bash
sudo nginx -t && sudo systemctl reload nginx
curl -I http://<VPS_IP>/
```

## 4. Grant `githubrunner` deploy permissions
**Run as:** sudo/root — one-time manual setup (grants `githubrunner` the permissions it needs for Section 6's automated deploys).
```bash
sudo chown -R githubrunner:githubrunner /var/www/taskmanagement
```
Allow `githubrunner` to restart the service without a password (sudoers drop-in):
```bash
echo 'githubrunner ALL=(ALL) NOPASSWD: /bin/systemctl restart taskmanagement.service, /bin/systemctl status taskmanagement.service' | sudo tee /etc/sudoers.d/githubrunner-taskmanagement
sudo visudo -c
```

## 5. Install the GitHub Actions self-hosted runner (org-level, shared across repos)
**Run as:** `githubrunner` — one-time manual setup (switch user first: `sudo su - githubrunner`).
Registered at the **organization** level so any repo in `Devnity-Rafal-Labecki` (task.management, future apps) can use this single runner via the `linux-vps` label — see Section 8 for how new repos opt in without reinstalling a runner.
```bash
mkdir -p ~/actions-runner && cd ~/actions-runner
RUNNER_VERSION=$(curl -s https://api.github.com/repos/actions/runner/releases/latest | grep -Po '"tag_name": "v\K[0-9.]+')
curl -o actions-runner-linux-x64.tar.gz -L "https://github.com/actions/runner/releases/download/v${RUNNER_VERSION}/actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz"
tar xzf actions-runner-linux-x64.tar.gz
```
Then register (replace `YOUR_ORG_TOKEN` with the value you copy from GitHub in the next step — do **not** leave angle brackets `< >` in the command, bash treats them as redirection operators):
```bash
./config.sh --url https://github.com/Devnity-Rafal-Labecki --token YOUR_ORG_TOKEN --labels linux-vps
```
Get the token from the **organization** (not the repo) — **Devnity-Rafal-Labecki → Settings → Actions → Runners → New runner** (requires org admin). The `--labels linux-vps` flag tags this runner so workflows target it with `runs-on: [self-hosted, linux-vps]`.

> **Troubleshooting `404 Not Found` on `POST .../actions/runner-registration`**: this is almost never a firewall issue (a firewall block shows as a timeout, not a clean 404 response from GitHub). Known causes:
> 1. **Token expired** — registration tokens are valid for only ~1 hour. Generate the token and run `./config.sh` immediately after; don't reuse an old one or let time pass between copying it and running the command.
> 2. **Account lacks org admin rights** — registering at the org level (`--url https://github.com/<org>`) requires admin access to the *organization*, not just the repo. If you only have repo access, either get org admin rights or fall back to a repo-level runner (`--url https://github.com/<org>/<repo>`, and drop the org-wide sharing from Section 8).
> 3. Re-generate a fresh token from the **org's** Settings → Actions → Runners page (not the repo's) and retry.

Install as a systemd service so it survives reboots — **exit back to your own sudo-capable admin user first** (`exit` out of the `sudo su - githubrunner` shell). `githubrunner` itself has no general sudo rights (only the narrow rule from Section 4), so running `sudo ./svc.sh ...` *as* `githubrunner` will be denied. `githubrunner`'s home directory is also mode `700` (owned by `githubrunner`), so a plain `cd` as another user fails with "Permission denied" — the `cd` must happen *inside* the `sudo` context (as root), not before it:
```bash
sudo bash -c "cd /home/githubrunner/actions-runner && ./svc.sh install githubrunner"
sudo bash -c "cd /home/githubrunner/actions-runner && ./svc.sh start"
sudo bash -c "cd /home/githubrunner/actions-runner && ./svc.sh status"
```
The `githubrunner` argument to `install` tells the script which user the *systemd service* should run as — it doesn't require you to be logged in as that user to run the command.

## 5b. Install the .NET SDK on the VPS (runtime alone isn't enough)
**Run as:** sudo/root — one-time manual setup.
The VPS only had the .NET **runtime** installed (needed to run `taskmanagement.service`), not the **SDK** — `dotnet restore/build/publish` in Section 6's workflow fail without it (the `Setup .NET` GitHub Action step was removed in Section 6 because it tried to install its own copy into `/usr/share/dotnet`, which `githubrunner` can't write to — installing the SDK system-wide via apt, as root, avoids that permission issue entirely).
```bash
wget https://packages.microsoft.com/config/ubuntu/$(lsb_release -rs)/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
rm packages-microsoft-prod.deb
sudo apt-get update
sudo apt-get install -y dotnet-sdk-10.0
dotnet --list-sdks
```
If `dotnet-sdk-10.0` isn't available yet for your Ubuntu release in Microsoft's feed, use the install script instead:
```bash
curl -sSL https://dot.net/v1/dotnet-install.sh -o dotnet-install.sh
sudo bash dotnet-install.sh --channel 10.0 --install-dir /usr/share/dotnet
dotnet --list-sdks
```

## 6. GitHub Actions workflow
**Run as:** `githubrunner` — this is the part that runs **automatically** on every push to `master` via the self-hosted runner installed in Section 5.

Implemented at `@c:\Projects\task.management\.github\workflows\deploy.yml`. The **health check step is currently commented out** (pending domain/SSL migration — the `curl -f https://localtasklist.com/` check would fail since the site is only reachable by bare IP over HTTP right now). Re-enable it once Section 9's SSL setup is done, ideally pointed at `http://127.0.0.1:5000` or the eventual HTTPS domain instead.
```yaml
name: Build and Deploy
on:
  push:
    branches: [master]

jobs:
  build-and-deploy:
    runs-on: [self-hosted, linux-vps]
    env:
      RELEASE_DIR: /var/www/taskmanagement/releases/${{ github.sha }}
      APP_ROOT: /var/www/taskmanagement
    steps:
      - uses: actions/checkout@v4

      - name: Restore
        run: dotnet restore Api.sln

      - name: Build
        run: dotnet build Api.sln --configuration Release --no-restore

      - name: Publish
        run: dotnet publish Api.Server/Api.Server.csproj --configuration Release --no-build --output "$RELEASE_DIR"
        # api.client is built automatically via the SPA MSBuild targets during publish

      - name: Atomic symlink swap
        run: |
          ln -sfn "$RELEASE_DIR" "$APP_ROOT/current_new"
          mv -Tf "$APP_ROOT/current_new" "$APP_ROOT/current"

      - name: Restart service
        run: sudo systemctl restart taskmanagement.service

      # - name: Health check
      #   run: |
      #     sleep 3
      #     curl -f https://localtasklist.com/ || (echo "Health check failed" && exit 1)

      - name: Cleanup old releases
        run: |
          cd "$APP_ROOT/releases"
          ls -1dt */ | tail -n +6 | xargs -r rm -rf
```
Notes:
- `ln -sfn ... && mv -Tf ...` makes the symlink swap atomic (no window where `current` is missing/half-updated).
- Keeps the last 5 releases for quick rollback; delete older ones.
- Adjust `dotnet-version` if the VPS runtime differs from `10.0.x`.

## 7. Remove old Jenkins-specific artifacts (optional cleanup)
**Run as:** you, manually (Jenkins UI on the old Windows box + repo edits) — not `githubrunner`.
- Delete/retire the Jenkins job on the old Windows box once GitHub Actions is verified working.
- Remove `@/c:\Projects\task.management\DevOps\AppOfflineFile` if it's no longer referenced (it was an IIS-specific `app_offline.htm` mechanism, not needed for the systemd/nginx setup).

## 8. Adding another site later (template)
**Run as:** sudo/root for steps 1-6 (systemd/nginx/permissions), `githubrunner` only executes step 7's workflow automatically afterward.
The org-level runner from Section 5 is **shared** — no new runner install needed. Once a second app (e.g. `okiki.ostroda`) needs to go on this same VPS, repeat this pattern:
1. Pick the next free Kestrel port (`5001`, `5002`, ...) and record it below.
2. Create `<app>.service` under `/etc/systemd/system/`, `WorkingDirectory`/`ExecStart` pointing at `/var/www/<app>/current`, `ASPNETCORE_URLS=http://127.0.0.1:<port>`.
3. Create `/var/www/<app>/releases/` + `current` symlink (same as Section 2).
4. Create `/etc/nginx/sites-available/<app>` with its own `server { listen 80; server_name <app-domain> www.<app-domain>; proxy_pass http://127.0.0.1:<port>; ... }` — **no** `default_server` (only one site may hold that, and it should stay on `taskmanagement` until its own domain is live, then move to whichever site should catch unmatched/bare-IP traffic, or replace with a generic "unrecognized host" block).
5. Symlink into `sites-enabled`, `sudo nginx -t && sudo systemctl reload nginx`.
6. Add a `chown githubrunner` + sudoers NOPASSWD restart line for `<app>.service` (Section 4 pattern).
7. Copy `.github/workflows/deploy.yml` into that repo, adjusting `RELEASE_DIR`, `APP_ROOT`, and the systemd service name — keep `runs-on: [self-hosted, linux-vps]` unchanged since it's the same shared org runner.

> Since the runner is shared across all org repos, any repo in `Devnity-Rafal-Labecki` can trigger workflows on this VPS. Consider a GitHub **runner group** scoped to only the repos that should have access if this becomes a concern.

**Port registry** (keep updated):
| App | Port | Service | Domain |
|---|---|---|---|
| taskmanagement | 5000 | taskmanagement.service | localtasklist.com (pending DNS) |
| *(next app)* | 5001 | — | — |

Since `localtasklist.com` isn't resolvable yet, testing a second site pre-DNS requires either `curl -H "Host: <app-domain>" http://YOUR_VPS_IP/` or a temporary `/etc/hosts` entry on your dev machine — bare-IP requests will keep hitting whichever site is `default_server`.

## 9. Domain migration follow-up (SSL) — do this once DNS points at the VPS
**Run as:** sudo/root — one-time manual setup.
```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d localtasklist.com -d www.localtasklist.com
```
Certbot will edit the `taskmanagement` nginx site to add a `listen 443 ssl` block and set up auto-renewal. Afterward, decide whether `taskmanagement` should keep `default_server`, or whether a dedicated catch-all block should handle unmatched Hosts once multiple domains exist. **Re-enable the health check step in Section 6's workflow** once this is done.

## 10. First test run & rollback plan
**Run as:** you (git push triggers it) → `githubrunner` (executes the deploy automatically) → sudo/root (manual rollback commands, if needed).
```bash
git commit --allow-empty -m "test: trigger CI deploy" && git push origin master
```
Watch the run in GitHub Actions UI, then on the VPS:
```bash
systemctl status taskmanagement.service
journalctl -u taskmanagement.service -n 50 --no-pager
curl -I http://YOUR_VPS_IP/
```
**Rollback**: point `current` at the previous release folder and restart:
```bash
ln -sfn /var/www/taskmanagement/releases/<previous-sha> /var/www/taskmanagement/current
sudo systemctl restart taskmanagement.service
```
