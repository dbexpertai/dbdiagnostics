# dbexpert Community Edition

dbexpert watches a database server, detects problems and names the root cause with the
evidence for it. It runs on your own server and connects to your database from there.

The Community Edition is free. One install monitors one database server. It supports SQL Server,
Oracle, PostgreSQL, MySQL and MariaDB, ClickHouse and MongoDB. IBM Db2 is supported by the
Windows install only.

This repository holds the downloads, the install steps and the [licence](LICENSE). The product
is proprietary; its source is not published.

## Download

| Platform | Requires | Download |
|---|---|---|
| Windows | Windows Server 2019, 2022 or 2025 | [dbexpert for Windows](https://app.dbexpert.ai/api/downloads/public/windows) (ISO, about 5 GB) |
| Ubuntu 24.04 LTS | x86_64 | [dbdiagnostics-ubuntu-x64.tar](https://github.com/dbexpertai/dbdiagnostics/releases/latest/download/dbdiagnostics-ubuntu-x64.tar) |
| Ubuntu 22.04 LTS | x86_64 | [dbdiagnostics-ubuntu-22.04-x64.tar](https://github.com/dbexpertai/dbdiagnostics/releases/latest/download/dbdiagnostics-ubuntu-22.04-x64.tar) |
| macOS | macOS 13 or later, Apple Silicon | [dbdiagnostics-macos-arm64.tar](https://github.com/dbexpertai/dbdiagnostics/releases/latest/download/dbdiagnostics-macos-arm64.tar) |
| Kubernetes | x86_64 nodes, Helm 3.14 or later, PostgreSQL 19 | [Helm chart](#kubernetes) |

Every release lists the checksums of its files in `SHA256SUMS.txt`. The Windows installer is
served by app.dbexpert.ai because it is larger than a GitHub release file may be.

## Install

Installing means accepting the [licence](LICENSE). The installers ask for it; `--accept-eula`
answers for an unattended run.

### Windows

1. Mount the ISO (double-click it).
2. Run `setup.exe` as Administrator. It installs its own PostgreSQL and a Windows service, and
   takes about two minutes.
3. Open `https://localhost:8081`.

### Ubuntu

```bash
mkdir -p ~/dbdiagnostics-install
tar -xf dbdiagnostics-ubuntu-x64.tar -C ~/dbdiagnostics-install
cd ~/dbdiagnostics-install && sudo ./setup-ubuntu.sh
```

On Ubuntu 22.04 use `dbdiagnostics-ubuntu-22.04-x64.tar`. Then open `https://localhost:8081`.

### macOS

```bash
mkdir -p ~/dbdiagnostics-install
tar -xf dbdiagnostics-macos-arm64.tar -C ~/dbdiagnostics-install
cd ~/dbdiagnostics-install && sudo ./setup-macos.sh
```

Then open `https://localhost:8081`.

### Kubernetes

There is no installer to download. A Helm chart runs the same Linux build as two workloads, the
console and the scheduler, against a PostgreSQL 19 database you provide.

1. Create the database and its role, once, as a superuser:

   ```sql
   CREATE ROLE dbexpert_adm LOGIN PASSWORD '...';
   CREATE DATABASE dbanalytics OWNER dbexpert_adm;
   ```

2. Install the chart. Its version is the release with the last part after a dash: release
   `v1.1.2.15` is chart `1.1.2-15`. The [latest release](https://github.com/dbexpertai/dbdiagnostics/releases/latest)
   shows the command with the current version.

   ```bash
   helm install dbd oci://ghcr.io/dbexpertai/charts/dbdiagnostics --version <chart version> \
     --namespace dbdiagnostics --create-namespace \
     --set postgresql.external.host=<your PostgreSQL host> \
     --set postgresql.external.password='...' \
     --set auth.jwtSecret="$(openssl rand -base64 48)"
   ```

   To try it without a database of your own, replace the two `postgresql.external` lines with
   `--set postgresql.embedded.enabled=true --set postgresql.embedded.password='...'`. That
   database has no backups and is for evaluation only.

3. Open the console:

   ```bash
   kubectl -n dbdiagnostics port-forward svc/dbd-dbdiagnostics 8081:8081
   # then open https://localhost:8081
   ```

4. Back up `taxonomy.key` from the install's volume. It is written once, and a backup of the
   database does not include it:

   ```bash
   kubectl -n dbdiagnostics exec deploy/dbd-dbdiagnostics-web -- cat /data/taxonomy.key
   ```

The full guide travels with the chart: ingress and TLS, upgrades, network policy, licensing per
cluster and air-gapped clusters.

```bash
helm show readme oci://ghcr.io/dbexpertai/charts/dbdiagnostics --version <chart version>
```

## First start

The console uses a self-signed certificate, so the browser warns on the first visit. You can
upload your own certificate later in Settings.

1. **Create the admin account.** The first visit asks for it.
2. **Activate.** The install asks for a community key, which is free and personal: it carries
   your email address. Type your email, and the code we send you; the install then fetches
   your key. On a server with no internet connection, get the key at
   [app.dbexpert.ai/activate](https://app.dbexpert.ai/activate) on any other device and paste it.
3. **Add your database** and wait for the first findings.

## Check the connections

Most installs that do not work cannot reach something. Ask the install what:

| Install | Command |
|---|---|
| In the console | Settings → License & cloud → Connection check |
| Ubuntu, macOS | `sudo /opt/dbexpertAI/bin/dbdiagnostics --connection-check` (macOS: `/usr/local/dbexpertAI/bin/dbdiagnostics`) |
| Kubernetes | `kubectl -n dbdiagnostics exec deploy/dbd-dbdiagnostics-web -- /usr/local/bin/entrypoint.sh connection-check` |

It tests the install's own database, app.dbexpert.ai and every monitored server: the name
resolves and the port answers. Each failed line says what to change. What usually fails depends
on where the install runs:

| The install runs | The database runs | What usually fails |
|---|---|---|
| in a cluster | in the same cluster | A network policy or service mesh in the database's namespace that does not admit the install's namespace. |
| in a cluster | in another cluster, or on a server | The cluster's DNS does not know the database's name; an egress firewall; a database in another cluster that is not exposed outside it. |
| on a server | anywhere | The server's firewall and its DNS. |

## What the install sends

**No sensitive data ever reaches the internet.** Nothing read from the databases you monitor
leaves your network in readable form: no content, no query text, none of the names of databases,
hosts, users or applications found there, and none of their credentials.

What the install does send is about the install itself, over HTTPS to app.dbexpert.ai:

- **Activation by email:** the email address you type, and the code. Pasting a key sends
  nothing at this step.
- **Registration to your dbexpert.ai account,** when the install can reach it: your key, the
  install's host name, its machine code, its version and operating system.
- **The licence check:** the install's own token.
- **The online monitor:** off until an administrator of the install turns it on. It then sends
  findings in which the names read from your systems have been replaced by stand-ins
  (`host-3f9q…`, `user-…`) before they leave; the table that maps stand-ins back to names stays
  on the install. Two things you typed yourself are sent as typed: the display name you gave a
  monitored server (unless it is a host name or an address) and, if you entered one, the address
  of the install's console on your network. Settings shows exactly what would be sent.

An install with no internet connection sends nothing and works: activate with a pasted key and
leave the online monitor off.

## Editions and support

The Community Edition is free for one database server per install. The Enterprise edition adds
the full feature set and a service level agreement; pricing is on request at
[dbexpert.ai](https://dbexpert.ai).

Questions and problems: [dbexpert.ai/contact](https://dbexpert.ai/contact).

## Licence

dbexpert is proprietary software of Uptimesoft Inc., doing business as dbexpert.ai. Use is
governed by the [End User License Agreement](LICENSE). Third-party components and their licences
are listed in `THIRD_PARTY_NOTICES.txt` inside every installer.
