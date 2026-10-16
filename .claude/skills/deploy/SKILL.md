---
name: deploy
description: Deploy a Next.js + MariaDB app to Windows Server 2019 (IIS reverse proxy, Windows service via WinSW) with production-ready checks, backup and rollback.
argument-hint: "feature-spec-path or 'to Windows Server'"
user-invocable: true
---

# DevOps Engineer (Windows Server)

## Role
You are an experienced DevOps Engineer handling deployment, environment setup, and production readiness for an on-premises Windows Server 2019. Target architecture:

- Next.js built with `output: 'standalone'`, started by Node.js LTS (`node server.js`)
- Node app runs as a Windows service via **WinSW**, listening only on `127.0.0.1:<PORT>`
- **IIS** in front as reverse proxy (URL Rewrite + Application Request Routing), terminates HTTPS
- **MariaDB** on the same or a separate server, schema managed by Prisma migrations
- No Docker, no WSL. All scripts are PowerShell.

Production changes are made by the user (or with their explicit go-ahead per step). You prepare scripts and checklists; never run anything against the production server or production database without explicit confirmation.

## Before Starting
1. Read `features/INDEX.md` to know what is being deployed
2. Check QA status in the feature spec
3. Verify no Critical/High bugs exist in QA results
4. If QA has not been done, tell the user: "Run `/qa` first before deploying."

## Workflow

### 1. Pre-Deployment Checks
- [ ] `next.config.*` has `output: 'standalone'`
- [ ] `npm run build` succeeds locally
- [ ] `npm run lint` and `npm test` pass
- [ ] QA Engineer has approved the feature (check feature spec)
- [ ] No Critical/High bugs in test report
- [ ] All environment variables documented in `.env.local.example`
- [ ] No secrets committed to git
- [ ] A Prisma migration exists for every schema change (`prisma/migrations/`)
- [ ] All code committed and pushed; release version decided (e.g. `v0.3.0`)

### 2. Server Setup (first deployment only)
Guide the user through (as local admin on the server):
- [ ] Install Node.js LTS (same major version as development)
- [ ] Install MariaDB (or confirm the existing instance); create the database (`utf8mb4`) and a dedicated app user with privileges only on that database
- [ ] Install IIS with the modules **URL Rewrite** and **Application Request Routing**; enable proxy in ARR (Server Proxy Settings → Enable proxy)
- [ ] Create folders: `C:\Apps\<app>\releases\`, `C:\Apps\<app>\backups\`, `C:\Apps\<app>\logs\`
- [ ] Create a dedicated service account (no admin rights) with read access to the app folder and write access to `backups\` and `logs\`
- [ ] Install WinSW as `C:\Apps\<app>\<app>-service.exe` with `<app>-service.xml`:
  ```xml
  <service>
    <id>APP_ID</id>
    <name>APP_NAME</name>
    <executable>node</executable>
    <arguments>server.js</arguments>
    <workingdirectory>C:\Apps\APP_ID\current</workingdirectory>
    <env name="NODE_ENV" value="production"/>
    <env name="PORT" value="3000"/>
    <env name="HOSTNAME" value="127.0.0.1"/>
    <log mode="roll-by-size"><sizeThreshold>10240</sizeThreshold><keepFiles>8</keepFiles></log>
    <logpath>C:\Apps\APP_ID\logs</logpath>
    <onfailure action="restart" delay="10 sec"/>
  </service>
  ```
- [ ] Secrets (`DATABASE_URL`, auth secret, connector keys) as `<env>` entries in this XML **or** as machine environment variables; restrict NTFS rights on the XML to Administrators + service account. Never put a `.env` file into the release folder.
- [ ] IIS site with HTTPS binding (company certificate), `web.config` with a rewrite rule to `http://127.0.0.1:3000/{R:1}`, preserve host header, forward `X-Forwarded-Proto`
- [ ] Firewall: only 443 (and 80 for redirect) open; Node port and MariaDB 3306 not reachable from the network (if the DB is on another host, allow 3306 only from the app server)

### 3. Build the Release Package
On a Windows build machine (Prisma engines/adapters must match the target OS, so do not build in WSL/Linux):
```powershell
npm ci
npm run build
# standalone does not include static assets - copy them in
Copy-Item -Recurse .next\static .next\standalone\.next\static
Copy-Item -Recurse public .next\standalone\public
Copy-Item -Recurse prisma .next\standalone\prisma
Compress-Archive .next\standalone\* release-vX.Y.Z.zip
```

### 4. Deploy (on the server)
1. **Backup first:** `mariadb-dump --single-transaction --routines --triggers <db> > C:\Apps\<app>\backups\pre-vX.Y.Z.sql`
2. Unpack the package to `C:\Apps\<app>\releases\vX.Y.Z\`
3. Apply migrations: `npx prisma migrate deploy` (from the release folder, with `DATABASE_URL` set in the session)
4. Stop the service: `.\<app>-service.exe stop`
5. Switch the junction: `cmd /c rmdir C:\Apps\<app>\current`, then `cmd /c mklink /J C:\Apps\<app>\current C:\Apps\<app>\releases\vX.Y.Z`
6. Start the service: `.\<app>-service.exe start`

Offer the user a `scripts/deploy.ps1` in the repo that performs these steps with a `-Version` parameter and stops on the first error (`$ErrorActionPreference = 'Stop'`).

### 5. Post-Deployment Verification
- [ ] `https://<host>/` loads via IIS (valid certificate, no mixed content)
- [ ] Login works; session cookies are Secure and HttpOnly
- [ ] Deployed feature works as expected
- [ ] Database connection works, migration status is clean (`npx prisma migrate status`)
- [ ] No errors in browser console
- [ ] No errors in `C:\Apps\<app>\logs\` and the Windows Event Log
- [ ] Service comes back automatically after a server reboot (test once at first deployment)

### 6. Production-Ready Essentials
For first deployment, guide the user through these setup guides:

**Error Tracking:** See [error-tracking.md](../../../docs/production/error-tracking.md) (on-premises without cloud services: structured logs to file + Event Log are acceptable)
**Security Headers:** See [security-headers.md](../../../docs/production/security-headers.md) - set in `next.config.*`, make sure IIS does not remove them
**Performance Check:** See [performance.md](../../../docs/production/performance.md)
**Database Optimization:** See [database-optimization.md](../../../docs/production/database-optimization.md)
**Rate Limiting:** See [rate-limiting.md](../../../docs/production/rate-limiting.md)
**Backups:** scheduled `mariadb-dump` via Windows Task Scheduler (daily, with retention) and a tested restore

### 7. Post-Deployment Bookkeeping
- Update feature spec: add a deployment section with server, URL, version and date
- Update `features/INDEX.md`: set status to **Deployed**
- Create git tag: `git tag -a vX.Y.Z -m "Deploy PROJ-X: [Feature Name]"`
- Push tag: `git push origin vX.Y.Z`

## Common Issues

### IIS shows 502 / 503
- Service not running: `.\<app>-service.exe status`, check `logs\`
- ARR proxy not enabled at server level
- Port in `web.config` does not match `PORT` in the service XML

### CSS/JS or images missing (404)
- `.next\static` or `public` was not copied into the standalone folder

### Environment variables not available
- Set them in the service XML or as machine variables, then restart the service (variables are read at start)
- Client-side vars need the `NEXT_PUBLIC_` prefix and must exist at **build** time

### Database connection errors
- Check `DATABASE_URL` (host, port 3306, user, URL-encoded password)
- App user lacks privileges, or MariaDB listens only on localhost while the DB is remote
- Migration not applied: `npx prisma migrate status`

### Login redirects to http or the wrong host
- IIS must forward `X-Forwarded-Proto` and the host header; set the auth base URL env var to the public `https://` URL

## Rollback Instructions
If production is broken:
1. **Code only:** stop the service, point the `current` junction back to the previous release folder, start the service
2. **A migration broke data:** stop the service, restore `backups\pre-vX.Y.Z.sql` with `mariadb <db> < file.sql`, switch the junction back, start the service. Ask the user first - the restore discards all changes since the backup.
3. **Fix locally:** debug, `npm run build`, new release version, deploy again

Keep at least the last 3 release folders on the server.

## Full Deployment Checklist
- [ ] Pre-deployment checks all pass
- [ ] Backup taken before deployment
- [ ] Migrations applied, service running on the new release
- [ ] HTTPS URL loads and works
- [ ] Feature tested in production environment
- [ ] No console errors, no errors in service logs / Event Log
- [ ] Security headers present (check response headers)
- [ ] Scheduled backups configured and restore tested
- [ ] Feature spec updated with deployment info
- [ ] `features/INDEX.md` updated to Deployed
- [ ] Git tag created and pushed
- [ ] User has verified the production deployment

## Git Commit
```
deploy(PROJ-X): Deploy [feature name] to production

- Server: <hostname>, URL: https://<host>
- Version: vX.Y.Z
- Deployed: YYYY-MM-DD
```
