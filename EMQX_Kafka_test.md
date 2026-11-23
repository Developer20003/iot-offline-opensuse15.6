# EMQX 5.8.9 + Apache Kafka 3.9.2 — Offline Second-Laptop Setup

**Target:** openSUSE Leap 15.6, Linux x86_64, no network access during installation or operation  
**Release repository:** [Developer20003/iot-offline-opensuse15.6 — Releases](https://github.com/Developer20003/iot-offline-opensuse15.6/releases)  
**Purpose:** unpack the artifacts already uploaded to GitHub on an internet-connected machine, transfer them by USB, start an MQTT broker and a single-node Kafka broker, and verify the EMQX web dashboard and message flow.

> **Important release distinction:** The Kafka release contains a runnable binary distribution (`kafka_2.13-3.9.2.tgz` split into two parts). The EMQX release advertises an **editable workspace** and a **cached Docker builder**. Those names do **not**, by themselves, guarantee a portable, ready-to-run EMQX release. Sections 4–5 explicitly inspect the workspace and provide a fallback. Do not confuse a builder image with a running broker image.

## 1. Releases and exact assets

Download these on the **internet-connected preparation laptop**, not on the disconnected laptop.

| Release | Essential files | Notes |
|---|---|---|
| [EMQX 5.8.9 offline](https://github.com/Developer20003/iot-offline-opensuse15.6/releases/tag/emqx-5.8.9-offline-v1) | `emqx589-editable-workspace.zip`, `README-OFFLINE.txt`, `SHA256SUMS.txt`, `SHA256-ORIGINALS.txt`, `versions.txt` | Source/compiled workspace; inspect for an actual broker executable |
| [Kafka 3.9.2 offline](https://github.com/Developer20003/iot-offline-opensuse15.6/releases/tag/kafka-3.9.2-offline-v1) | `kafka_2.13-3.9.2.tgz.part-000`, `kafka_2.13-3.9.2.tgz.part-001`, `SHA256SUMS-parts.txt`, `SHA256-ORIGINALS.txt` | Required binary distribution parts |
| [openSUSE 15.6 RPM bundle](https://github.com/Developer20003/iot-offline-opensuse15.6/releases/tag/leap15.6-iot-v1) | `opensuse-leap15.6-iot-rpms-x86_64.zip` | Optional system package pool: Java 17, tools, dependencies; do not install all RPMs blindly |
| [EMQX Dashboard source](https://github.com/Developer20003/iot-offline-opensuse15.6/releases/tag/emqx-dashboard-v1.10.5-offline-v1) | Six `emqx-dashboard5-v1.10.5-offline.zip.part-*` files plus checksums | **Optional development source**; a complete EMQX runtime should serve its own dashboard at `:18083` |

For **EMQX source compilation**, the optional `emqx589-builder-cached-linux-amd64.tar.gz` is also in the EMQX release, but it is a cached **builder image**, not verified to be a runnable MQTT broker image. A Docker daemon and its own offline packages would be required to use it.

For **Kafka source development**, the source tarball and five `kafka-3.9.2-gradle-home.tar.gz.part-*` files are optional. **They are not required to run the broker.**

### Download the required release assets on the first (online) laptop

If you prefer a terminal over the GitHub Releases web page, run this **only on the online machine**:

```bash
BASE=https://github.com/Developer20003/iot-offline-opensuse15.6/releases/download
mkdir -p offline-usb/emqx offline-usb/kafka
for file in emqx589-editable-workspace.zip README-OFFLINE.txt SHA256SUMS.txt SHA256-ORIGINALS.txt versions.txt; do
  curl -fL --retry 3 -o "offline-usb/emqx/$file" \
    "$BASE/emqx-5.8.9-offline-v1/$file" || exit 1
done
for file in kafka_2.13-3.9.2.tgz.part-000 kafka_2.13-3.9.2.tgz.part-001 SHA256SUMS-parts.txt SHA256-ORIGINALS.txt; do
  curl -fL --retry 3 -o "offline-usb/kafka/$file" \
    "$BASE/kafka-3.9.2-offline-v1/$file" || exit 1
done
```

The EMQX workspace is about **383 MB**; the two Kafka binary parts total about **122 MB**. Copy the `offline-usb` directory onto a USB stick. This is not a substitute for producing a compiled EMQX runtime if the workspace does not include one (see Section 6).

### Arrange the USB drive

```text
USB_DRIVE/
  emqx/
    emqx589-editable-workspace.zip
    README-OFFLINE.txt
    SHA256SUMS.txt
    SHA256-ORIGINALS.txt
    versions.txt
  kafka/
    kafka_2.13-3.9.2.tgz.part-000
    kafka_2.13-3.9.2.tgz.part-001
    SHA256SUMS-parts.txt
    SHA256-ORIGINALS.txt
  rpms/                              # Optional system RPMs
```

Transfer **all the required parts** using a USB drive. Do not attempt `wget`, `curl`, `git clone`, `zypper refresh`, or `npm install` on the disconnected laptop.

## 2. Prerequisites and safety checks on the second laptop

Run the initial setup commands in a root terminal or prefix filesystem operations with `sudo`. **Use a non-root service user for a permanent deployment.**

```bash
cat /etc/os-release | grep -E '^(NAME|VERSION_ID|PRETTY_NAME)='
uname -m
java -version
command -v tar
command -v unzip || echo 'unzip is missing: install its RPM from your offline pool first'
command -v sha256sum
ss -ltnp | grep -E ':1883|:18083|:8080|:9092|:9093' || true
```

Expected: openSUSE Leap **15.6**, architecture `x86_64`, and **Java 17** available. Kafka 3.9.2 runs with Java 17. If Java, `unzip`, or another required system tool is absent, use the offline RPM pool; use the package manager to resolve dependencies, **never `rpm --nodeps`**. The release asset lists do not establish that every dependency will already be installed on the second laptop.

**MQTT port collision:** ThingsBoard CE normally binds MQTT on **1883** as well. If ThingsBoard is running and owns that port, do not start EMQX on the same address/port. Prefer EMQX on **1883** and configure ThingsBoard to use **1884** (`MQTT_BIND_PORT=1884` in the ThingsBoard service environment) or disable ThingsBoard's MQTT transport if your planned architecture uses EMQX as the only MQTT ingress. Restart ThingsBoard only after checking its current configuration. The ThingsBoard web UI stays on **8080**.

## 3. Copy and verify assets offline

Set `USB` to the **actual mount point** displayed by your file manager or `lsblk -f`:

```bash
USB=/run/media/YOUR_USERNAME/YOUR_USB_LABEL
ROOT=/opt/iot-offline-opensuse15.6
mkdir -p "$ROOT/offline-install-assets/emqx" "$ROOT/offline-install-assets/kafka"
cp -av "$USB/emqx/." "$ROOT/offline-install-assets/emqx/"
cp -av "$USB/kafka/." "$ROOT/offline-install-assets/kafka/"
```

Verify Kafka archive parts **before concatenating**:

```bash
cd /opt/iot-offline-opensuse15.6/offline-install-assets/kafka || exit 1
grep -F 'kafka_2.13-3.9.2.tgz.part-' SHA256SUMS-parts.txt | sha256sum -c - || exit 1
cat kafka_2.13-3.9.2.tgz.part-000 kafka_2.13-3.9.2.tgz.part-001 > kafka_2.13-3.9.2.tgz
# Check the reconstructed original against the uploaded original checksum:
grep -F 'kafka_2.13-3.9.2.tgz' SHA256-ORIGINALS.txt | sha256sum -c - || exit 1
```

Verify EMQX against the uploaded checksum file:

```bash
cd /opt/iot-offline-opensuse15.6/offline-install-assets/emqx || exit 1
grep -F 'emqx589-editable-workspace.zip' SHA256SUMS.txt | sha256sum -c - || exit 1
```

If any `grep` produces **no lines**, inspect the checksum file with `cat` rather than assuming verification succeeded. The checksum files may use a different filename format. If `sha256sum` reports **FAILED**, do not extract that artifact; recopy it from USB.

## 4. Extract and start Apache Kafka 3.9.2 (KRaft, one laptop)

**Extract the binary, not the source archive:**

```bash
ROOT=/opt/iot-offline-opensuse15.6
mkdir -p "$ROOT/runtime"
tar -xzf "$ROOT/offline-install-assets/kafka/kafka_2.13-3.9.2.tgz" -C "$ROOT/runtime"
KAFKA="$ROOT/runtime/kafka_2.13-3.9.2"
test -f "$KAFKA/bin/kafka-server-start.sh" || { echo 'Kafka runtime not found'; exit 1; }
```

Create a **dedicated persistent data directory** and a separate configuration file. The configuration in this guide is for a **new single-node development cluster**, not a production cluster:

```bash
ROOT=/opt/iot-offline-opensuse15.6
KAFKA="$ROOT/runtime/kafka_2.13-3.9.2"
mkdir -p "$ROOT/runtime/kafka-data" "$ROOT/runtime/kafka-config"
if [ -f "$KAFKA/config/kraft/reconfig-server.properties" ]; then
  BASE="$KAFKA/config/kraft/reconfig-server.properties"
else
  BASE="$KAFKA/config/kraft/server.properties"
fi
CFG="$ROOT/runtime/kafka-config/server.properties"
if [ ! -f "$CFG" ]; then
  cp "$BASE" "$CFG"
  if grep -q '^log.dirs=' "$CFG"; then
    sed -i "s|^log.dirs=.*|log.dirs=$ROOT/runtime/kafka-data|" "$CFG"
  else
    printf '\nlog.dirs=%s/runtime/kafka-data\n' "$ROOT" >> "$CFG"
  fi
fi
printf 'Kafka configuration: %s\n' "$CFG"
grep -E '^(process.roles|node.id|listeners|controller.quorum|log.dirs)=' "$CFG" || true
```

**One-time format only for a new, empty Kafka cluster.** Formatting a previously initialized cluster can damage its metadata or make stored messages inaccessible. Check before running:

```bash
ROOT=/opt/iot-offline-opensuse15.6
KAFKA="$ROOT/runtime/kafka_2.13-3.9.2"
CFG="$ROOT/runtime/kafka-config/server.properties"
if [ -f "$ROOT/runtime/kafka-data/meta.properties" ]; then
  echo 'Kafka storage already formatted: SKIP FORMAT'
else
  test -z "$(ls -A "$ROOT/runtime/kafka-data")" || {
    echo 'Kafka data dir is NOT empty; inspect before formatting'; exit 1;
  }
  CLUSTER_ID=$("$KAFKA/bin/kafka-storage.sh" random-uuid) || exit 1
  if grep -q '^controller.quorum.voters=' "$CFG"; then
    "$KAFKA/bin/kafka-storage.sh" format -t "$CLUSTER_ID" -c "$CFG" || exit 1
  else
    "$KAFKA/bin/kafka-storage.sh" format --standalone -t "$CLUSTER_ID" -c "$CFG" || exit 1
  fi
fi
```

Start Kafka in the **current terminal** so that startup errors remain visible:

```bash
ROOT=/opt/iot-offline-opensuse15.6
KAFKA="$ROOT/runtime/kafka_2.13-3.9.2"
CFG="$ROOT/runtime/kafka-config/server.properties"
export KAFKA_HEAP_OPTS='-Xms512m -Xmx1g'
"$KAFKA/bin/kafka-server-start.sh" "$CFG"
```

Open a **second terminal** to check it:

```bash
ROOT=/opt/iot-offline-opensuse15.6
KAFKA="$ROOT/runtime/kafka_2.13-3.9.2"
ss -ltnp | grep ':9092' || true
"$KAFKA/bin/kafka-broker-api-versions.sh" --bootstrap-server 127.0.0.1:9092
"$KAFKA/bin/kafka-topics.sh" --bootstrap-server 127.0.0.1:9092 --list
```

**Kafka message round-trip smoke test:**

```bash
ROOT=/opt/iot-offline-opensuse15.6
KAFKA="$ROOT/runtime/kafka_2.13-3.9.2"
"$KAFKA/bin/kafka-topics.sh" --bootstrap-server 127.0.0.1:9092 \
  --create --if-not-exists --topic offline.test --partitions 1 --replication-factor 1
printf 'offline-kafka-test-%s\n' "$(date +%s)" | \
  "$KAFKA/bin/kafka-console-producer.sh" --bootstrap-server 127.0.0.1:9092 --topic offline.test
"$KAFKA/bin/kafka-console-consumer.sh" --bootstrap-server 127.0.0.1:9092 \
  --topic offline.test --from-beginning --timeout-ms 10000
```

The consumer should print the test message and may exit with a timeout after consuming all available records; that timeout is not a failure if the message appeared. On later starts, **do not repeat storage formatting**. Restart Kafka using the same `CFG` and `kafka-data` directory. To stop the foreground broker, press `Ctrl+C` in its terminal. For a persistent deployment, install a systemd service running as a dedicated non-root user.

## 5. Extract and start EMQX 5.8.9

Unpack the **editable workspace**, then locate an **actual compiled broker release**:

```bash
ROOT=/opt/iot-offline-opensuse15.6
EMQX_ASSETS="$ROOT/offline-install-assets/emqx"
EMQX_WORK="$ROOT/runtime/emqx589-workspace"
mkdir -p "$EMQX_WORK"
unzip -q "$EMQX_ASSETS/emqx589-editable-workspace.zip" -d "$EMQX_WORK"

# Find a compiled Erlang release script, NOT just source files.
find "$EMQX_WORK" -path '*/rel/emqx/bin/emqx' -print
# Also look for ready-made runtime tarballs, if present:
find "$EMQX_WORK" -type f \( -iname '*emqx*.tar.gz' -o -iname '*emqx*.tgz' \) -print | head -20
```

**If the first `find` prints a path** like `.../_build/emqx/rel/emqx/bin/emqx`, attempt to start that release from its own installation directory:

```bash
ROOT=/opt/iot-offline-opensuse15.6
EMQX_WORK="$ROOT/runtime/emqx589-workspace"
EMQX_BIN=$(find "$EMQX_WORK" -path '*/rel/emqx/bin/emqx' -print -quit)
test -n "$EMQX_BIN" && test -f "$EMQX_BIN" || {
  echo 'NO COMPILED EMQX RELEASE FOUND. See section 6; do not treat the builder as a broker.'; exit 1;
}
EMQX_HOME="${EMQX_BIN%/bin/emqx}"
cd "$EMQX_HOME" || exit 1
chmod +x ./bin/emqx
./bin/emqx start
./bin/emqx ctl status
./bin/emqx ctl listeners
```

For a foreground run (useful for debugging), use `./bin/emqx console` **instead of** `start`. Stop with `./bin/emqx stop` after daemon startup, or `Ctrl+C` in console mode. A compiled workspace may contain absolute build paths or native dependencies; if it fails after relocation, prepare a proper release on the first laptop rather than altering the second laptop's source tree blindly.

If the workspace contains a separately packaged **EMQX runtime tarball**, extract it according to its own layout, `cd` to the directory containing `bin/emqx`, then run `./bin/emqx start` and `./bin/emqx ctl status`.

### EMQX web UI

On the **second laptop's browser**, open:

- EMQX Dashboard: **http://127.0.0.1:18083/**
- Default first-install credentials (only if not previously changed): `admin` / `public`
- **Change the password immediately** and create MQTT device credentials before connecting production devices.

A complete broker runtime normally contains the compiled Dashboard. The separate `emqx-dashboard5-v1.10.5-offline.zip.part-*` release is **frontend development material**, not a standalone EMQX broker or a preconfigured server.

Check listeners and dashboard:

```bash
ss -ltnp | grep -E ':1883|:18083' || true
curl -I --max-time 5 http://127.0.0.1:18083/ || true
```

## 6. What to do if EMQX has no compiled `bin/emqx`

The uploaded EMQX assets include `emqx589-editable-workspace.zip` and `emqx589-builder-cached-linux-amd64.tar.gz`. The **builder image is not evidence of a runnable EMQX broker**. If the workspace does not contain `*/rel/emqx/bin/emqx` or a runtime tarball, you must **prepare and transfer the broker runtime on the first laptop**.

On the **first laptop** (where EMQX was successfully compiled), find its compiled release:

```bash
ROOT=/opt/iot-offline-opensuse15.6
find "$ROOT/repos/02-erlang-emqx" -path '*/rel/emqx/bin/emqx' -print
```

If there is a working compiled release, create a **tar archive that preserves executable permissions and symlinks**. Replace `WORKING_EMQX_RELEASE_DIRECTORY` with the directory that directly contains `bin/emqx`:

```bash
cd /path/to/WORKING_EMQX_RELEASE_DIRECTORY/.. || exit 1
tar -czf /path/to/USB_DRIVE/emqx-5.8.9-runtime-opensuse15.6-x86_64.tar.gz \
  WORKING_EMQX_RELEASE_DIRECTORY
sha256sum /path/to/USB_DRIVE/emqx-5.8.9-runtime-opensuse15.6-x86_64.tar.gz \
  > /path/to/USB_DRIVE/emqx-5.8.9-runtime.sha256
```

Then extract the runtime tarball on the second laptop and follow Section 5. If no working compiled release exists, **build EMQX on the first laptop using its cached toolchain and the release's `README-OFFLINE.txt`**, verify the broker starts there, and only then export the runtime. Do not assume that the source ZIP or builder image alone can serve MQTT traffic. If the compiled build is not portable between the two installations, compile on the second laptop using an already-transferred offline build toolchain and matching OS packages.

For offline Docker-based builds, `docker load -i emqx589-builder-cached-linux-amd64.tar.gz` **loads the builder image only**; it neither starts EMQX nor automatically creates a broker image. The Docker engine and any needed system RPMs must also be present offline.

## 7. UI and connectivity summary

| Service | Default address | What should appear |
|---|---|---|
| EMQX Dashboard | `http://127.0.0.1:18083` | Broker's built-in web UI; Clients, Topics, Listeners |
| EMQX MQTT | `127.0.0.1:1883` | MQTT TCP listener, test via an MQTT client |
| Kafka | `127.0.0.1:9092` | **No built-in web UI**; verify with Kafka CLI producer and consumer |
| ThingsBoard (separately installed) | `http://127.0.0.1:8080` | ThingsBoard login and dashboards |
| Node.js User Management (separate project) | `http://127.0.0.1:3000` | User UI/integration checks when installed; **not part of these releases** |

**EMQX and Kafka do not automatically communicate with each other.** The steps above verify that each runs independently. A Node.js producer/consumer or a supported EMQX Kafka integration must be implemented and configured for `EMQX -> Kafka -> ThingsBoard` data flow. Kafka's own topic and API checks do not demonstrate that integration.

## 8. Common errors and safe troubleshooting

| Symptom | Likely cause | Action |
|---|---|---|
| `Address already in use :1883` | ThingsBoard MQTT or another broker already owns the port | Run `ss -ltnp`; assign different MQTT bind ports before restarting the relevant service |
| `Could not find or load main class` or `java: command not found` | Java unavailable or wrong JVM/PATH | Install/activate Java 17 from offline RPM pool; check `java -version` |
| Kafka `No readable meta.properties` | Unformatted fresh KRaft storage, or wrong `log.dirs` | Inspect `CFG` and storage; **format only an empty first-run directory** |
| Kafka `InconsistentClusterIdException` | Cluster ID/metadata mismatch | Stop and inspect persistent data; never blindly reformat data |
| Kafka `Connection to node ... could not be established` | Wrong port, listener, or advertised address | Inspect `listeners`, `advertised.listeners`, port 9092, and broker logs |
| EMQX `bin/emqx` missing | Uploaded workspace is not a broker release | Follow Section 6; produce a runnable EMQX tarball on the first laptop |
| EMQX `lib*.so` not found | Missing native library or runtime incompatibility | Determine needed libraries with `ldd` on relevant binaries; acquire matching RPMs on first laptop |
| EMQX page on `18083` unavailable | Broker not started or dashboard listener disabled | Check `./bin/emqx ctl status`, `./bin/emqx ctl listeners`, and process logs |
| GitHub Release parts fail SHA-256 | Incomplete or corrupted USB transfer | Recopy part(s) on the preparation machine, verify again before extract |
| Kafka CLI works but EMQX message not in Kafka | No bridge configured | This is expected until a bridge/connector is installed and enabled |

Useful diagnostics:

```bash
ss -ltnp | grep -E ':1883|:18083|:8080|:9092|:9093' || true
ps -ef | grep -E '[b]eam.smp|[k]afka.Kafka' || true
journalctl -xe --no-pager | tail -80
```

## 9. Offline handoff checklist

- [ ] All required release parts downloaded on **online** machine.
- [ ] USB copied to second laptop; checksums of parts and reconstructed Kafka archive verified.
- [ ] openSUSE Leap 15.6 x86_64 confirmed; Java 17, tar, unzip present.
- [ ] No unresolved 1883 port collision with ThingsBoard.
- [ ] Kafka extracted from **binary** archive, data directory initialized once, broker listening on 9092.
- [ ] Kafka topic creation and produce/consume message round trip passed.
- [ ] EMQX **compiled runtime** located and successfully started (or prepared separately on first laptop).
- [ ] EMQX port 1883 and Dashboard port 18083 reachable.
- [ ] EMQX dashboard administrator password changed; test MQTT client connected.
- [ ] Kafka and EMQX persistent data paths backed up and maintained across restarts.
- [ ] Optional: install the separate Node.js User Management project for UI-based integration checks.

**Reference docs (online preparation machine only):**

- [Apache Kafka 3.9 Quick Start](https://kafka.apache.org/39/getting-started/quickstart/)
- [Apache Kafka 3.9 KRaft](https://kafka.apache.org/39/operations/kraft/)
- [EMQX deployment documentation](https://docs.emqx.com/en/emqx/latest/get-started/getting-started.html)
- [ThingsBoard MQTT transport configuration](https://thingsboard.io/docs/reference/configuration/mqtt-transport-config/)
