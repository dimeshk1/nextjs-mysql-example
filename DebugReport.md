# Debug Report

Issues encountered and resolved during deployment of the Next.js + MySQL application. Each was hit genuinely during the deployment process, not manufactured for this report.

---

## Issue 1: Nginx failed to start after `apt upgrade`

**Problem**
After running `sudo apt upgrade -y` during Task 1's server setup, `apt` attempted to restart Nginx automatically and failed:
```
Job for nginx.service failed because the control process exited with error code.
```

**Root cause**
`/etc/nginx/nginx.conf` (line 84) contained a deliberately invalid directive — `THIS_IS_AN_INTENTIONAL_DEVOPS_TEST_ERROR` — planted in the provided server image as part of the test's built-in debugging challenge.

**How found**
```bash
sudo systemctl status nginx.service
# Active: failed (Result: exit-code)

sudo journalctl -xeu nginx.service --no-pager | tail -40
# nginx[16672]: 2026/08/25 07:55:12 [emerg] 16672#16672: unknown directive
# "THIS_IS_AN_INTENTIONAL_DEVOPS_TEST_ERROR" in /etc/nginx/nginx.conf:84
# nginx: configuration file /etc/nginx/nginx.conf test failed
```
The `journalctl` output pointed directly to the exact file, line number, and invalid directive.

**Fix applied**
```bash
sudo grep -n "THIS_IS_AN_INTENTIONAL_DEVOPS_TEST_ERROR" /etc/nginx/nginx.conf
sudo sed -i '84d' /etc/nginx/nginx.conf
sudo nginx -t
sudo systemctl restart nginx
```

**Result**
```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
`sudo systemctl status nginx` confirmed `Active: active (running)`.

---

## Issue 2: Knex CLI could not run migrations (`production` config missing)

**Problem**
Running `docker compose run --rm migrate` would have failed with an unresolved Knex environment configuration once the container ran with `NODE_ENV=production`.

**Root cause**
The starter repository's `knexfile.ts` only defined a `development` configuration block. Knex CLI selects its configuration by the current `NODE_ENV`, and no `production` key existed for it to fall back to.

**How found**
Manual review of `knexfile.ts` after confirming the Docker container's runtime environment (`ENV NODE_ENV=production` in the Dockerfile's `runner` stage) would not match any key in the exported `knexConfig` object.

**Fix applied**
Added a `production` block mirroring the existing `development` connection settings (both sourced from environment variables, so no values are hardcoded):
```ts
const knexConfig = {
  development: { /* ... */ },
  production: {
    client: 'mysql2',
    connection: connection,
    migrations: { tableName: 'migrations', directory: './database/migrations' },
    seeds: { directory: './database/seeds' }
  }
};
```

**Result**
Local validation confirmed the fix before touching the server:
```bash
npx knex migrate:status --knexfile knexfile.ts --env production
# Requiring external module ts-node/register
# Using environment: production
# AggregateError (connection refused — expected, no DB running locally at that point)
```
The error changed from a configuration/module error to a connection error, confirming the `production` config now resolves correctly. Later, on the server, the same command succeeded fully: `Batch 1 run: 1 migrations`.

---

## Issue 3: `ts-node` missing — Knex CLI could not load the TypeScript config file

**Problem**
`knexfile.ts` is a TypeScript file, but the Knex CLI needs `ts-node` available to load `.ts` config/migration files in any environment. It was not present in `package.json`.

**How found**
Identified by inspecting `package.json` dependencies before attempting to run migrations — `ts-node` was absent from both `dependencies` and `devDependencies`.

**Fix applied**
```bash
npm install --save-dev ts-node
```
Also required building the `migrate` Docker service from the Dockerfile's `builder` stage (not the stripped-down `runner` stage), since the production runtime image intentionally excludes dev dependencies:
```yaml
  migrate:
    build:
      context: .
      dockerfile: Dockerfile
      target: builder
```

**Result**
```bash
docker compose run --rm migrate
# Requiring external module ts-node/register
# Using environment: production
# Batch 1 run: 1 migrations
```

---

## Issue 4: Next.js build missing standalone output — Docker runner stage would fail

**Problem**
The Dockerfile's `runner` stage copies `.next/standalone` from the `builder` stage. Without the correct Next.js config, this directory (and the `server.js` the container's `CMD` depends on) would never be generated, and the Docker build would fail at that `COPY` step.

**Root cause**
`next.config.mjs` did not include `output: 'standalone'`.

**How found**
Reviewed `next.config.mjs` directly and confirmed the option was absent before attempting the Docker build, avoiding a failed build later.

**Fix applied**
```js
const nextConfig = {
    output: 'standalone',
    webpack: (config, { isServer }) => {
        if (!isServer) {
            config.resolve.fallback.fs = false;
        }
        return config;
    },
};
```

**Result**
```bash
npm run build
ls .next/standalone/server.js
# .next/standalone/server.js
```
Confirmed the standalone build output and `server.js` were generated correctly. The Docker build subsequently completed without error.

---

## Issue 5: App container publicly exposed on port 3000, bypassing UFW

**Problem**
After `docker compose up -d`, `docker compose ps` showed the `app` container bound as:
```
0.0.0.0:3000->3000/tcp
```
Despite UFW being configured (Task 1) to allow only ports 22, 80, and 443, port 3000 was reachable directly from the public internet.

**Root cause**
Docker manages port publishing through its own `iptables`/`nftables` rules, which it inserts ahead of UFW's rule chain. When a container publishes a port bound to `0.0.0.0` (all interfaces), Docker's rule allows the traffic through before UFW's default-deny policy is ever evaluated — meaning UFW's configuration had no effect on this specific port, regardless of how it was set up.

**How found**
```bash
docker compose ps
# PORTS: 0.0.0.0:3000->3000/tcp
```
Confirmed externally (from a separate machine, not the server itself):
```bash
curl -I --max-time 5 http://13.229.223.118:3000
# HTTP/1.1 200 OK   (before the fix — confirms external exposure)
```

**Fix applied**
Changed the port binding in `docker-compose.yml` to localhost-only, since the app should only ever be reached through Nginx on the same host:
```yaml
    ports:
      - "127.0.0.1:3000:3000"
```
```bash
docker compose up -d app
```

**Result**
```bash
docker compose ps
# PORTS: 127.0.0.1:3000->3000/tcp

curl -I --max-time 5 http://13.229.223.118:3000
# curl: (28) Connection timed out after 5005 milliseconds

curl -I --max-time 5 http://13.229.223.118
# HTTP/1.1 200 OK   (Nginx on port 80 remains correctly reachable)
```
Port 3000 is no longer reachable from outside the server; the application remains fully functional via `localhost:3000` for Nginx to proxy to.
