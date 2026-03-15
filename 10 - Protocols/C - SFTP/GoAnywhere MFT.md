# SFTP — Secure File Transfer Protocol

**Tags:** #sftp #ssh #file-transfer #security #paramiko #goanywhere #mft #data-engineering

> Comprehensive reference for SFTP in data engineering: protocol fundamentals, key management, scripted transfers, managed file transfer platforms, and landing zone patterns.

---

## 1. SFTP vs FTP vs FTPS — Protocol Comparison

| Feature | FTP | FTPS | SFTP |
|---|---|---|---|
| **Full Name** | File Transfer Protocol | FTP over SSL/TLS | SSH File Transfer Protocol |
| **Underlying Protocol** | TCP (plain text) | TCP + TLS | SSH |
| **Default Port** | 21 (control) + dynamic data | 21 / 990 (implicit) | 22 |
| **Encryption** | None | TLS for control + data channels | Full SSH encryption |
| **Authentication** | Username / password | Username / password + certificates | Password, SSH keys, certificates |
| **Firewall Friendliness** | Poor — requires dynamic data ports | Poor — same as FTP | Excellent — single port |
| **NAT Traversal** | Problematic (active/passive modes) | Problematic | Straightforward |
| **Certificate Management** | N/A | PKI / CA certificates | SSH key pairs |
| **Compliance Suitability** | Not recommended | Acceptable with TLS 1.2+ | Preferred for most standards |
| **Data Engineering Usage** | Legacy only | Some enterprise integrations | Industry standard |

> [!tip] Prefer SFTP for new integrations. It uses a single port, avoids the complexity of FTP's dual-channel architecture, and is natively supported by every major operating system and cloud platform.

---

## 2. SFTP Architecture

### SSH Transport Layer

SFTP operates as a subsystem of [[SSH]]. The connection lifecycle:

1. **TCP handshake** on port 22
2. **Key exchange** — Diffie-Hellman or ECDH to establish shared secret
3. **Server authentication** — client verifies server's host key against `known_hosts`
4. **User authentication** — password, public key, or certificate
5. **Channel establishment** — SFTP subsystem request over encrypted channel
6. **File operations** — open, read, write, stat, rename, remove, mkdir, etc.

### Authentication Methods

| Method | Security Level | Use Case |
|---|---|---|
| **Password** | Basic | Interactive use, initial setup |
| **Public Key** | Strong | Automated transfers, service accounts |
| **Keyboard-Interactive** | Variable | Multi-factor authentication |
| **Certificate-Based** | Strongest | Enterprise / large-scale deployments |
| **GSSAPI / Kerberos** | Strong | Active Directory-integrated environments |

**Public key authentication flow:**

```
Client                              Server
  |--- SSH connection request -------->|
  |<-- Server host key + challenge ----|
  |--- Signed challenge (private key)->|
  |    Server checks authorised_keys   |
  |<-- Authentication success ---------|
```

---

## 3. Key Management

### Generating Key Pairs with ssh-keygen

```bash
# Generate Ed25519 key (recommended — fast, secure, short keys)
ssh-keygen -t ed25519 -C "data-pipeline@company.com" -f ~/.ssh/id_pipeline

# Generate RSA 4096-bit key (wider compatibility)
ssh-keygen -t rsa -b 4096 -C "sftp-service@company.com" -f ~/.ssh/id_sftp_rsa

# Generate key without passphrase (for automated pipelines)
ssh-keygen -t ed25519 -f ~/.ssh/id_automated -N ""

# View public key fingerprint
ssh-keygen -lf ~/.ssh/id_pipeline.pub
```

### Key Files and Their Roles

| File | Location | Purpose |
|---|---|---|
| `id_ed25519` | `~/.ssh/` | Client private key — never share |
| `id_ed25519.pub` | `~/.ssh/` | Client public key — copy to server |
| `authorized_keys` | `~/.ssh/` (server) | Lists permitted public keys |
| `known_hosts` | `~/.ssh/` (client) | Cached server host key fingerprints |
| `ssh_config` | `~/.ssh/config` | Per-host connection settings |
| `sshd_config` | `/etc/ssh/` | Server-side SSH daemon configuration |

### Managing authorised_keys

```bash
# Copy public key to remote server
ssh-copy-id -i ~/.ssh/id_pipeline.pub user@sftp-server.example.com

# Manual approach
cat ~/.ssh/id_pipeline.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh/

# Restrict key to SFTP-only (in authorized_keys)
command="internal-sftp",restrict ssh-ed25519 AAAA... data-pipeline@company.com
```

### Managing known_hosts

```bash
# Fetch server fingerprint before first connection
ssh-keyscan -t ed25519 sftp-server.example.com >> ~/.ssh/known_hosts

# Remove stale entry (after server rebuild)
ssh-keygen -R sftp-server.example.com

# Hash known_hosts for privacy
ssh-keygen -H -f ~/.ssh/known_hosts
```

### SSH Config for Multiple SFTP Endpoints

```
# ~/.ssh/config
Host vendor-sftp
    HostName sftp.vendor.example.com
    User svc_ingest
    Port 22
    IdentityFile ~/.ssh/id_vendor
    StrictHostKeyChecking yes

Host client-sftp
    HostName sftp.client.example.com
    User data_export
    Port 2222
    IdentityFile ~/.ssh/id_client
    ServerAliveInterval 60
    ServerAliveCountMax 3
```

---

## 4. Common SFTP Commands

### Interactive sftp Client

```bash
# Connect
sftp user@host
sftp -P 2222 user@host                       # Non-standard port
sftp -i ~/.ssh/id_pipeline user@host          # Specific key

# Common session commands
sftp> ls                                      # List remote files
sftp> lls                                     # List local files
sftp> cd /inbound/orders                      # Change remote directory
sftp> lcd /data/landing                       # Change local directory
sftp> get report_20260315.csv                 # Download single file
sftp> mget *.csv                              # Download multiple files
sftp> put export_20260315.parquet             # Upload single file
sftp> mput *.parquet                          # Upload multiple files
sftp> mkdir /outbound/processed               # Create remote directory
sftp> rename file.csv file.csv.done           # Rename remote file
sftp> rm /inbound/old_file.csv                # Delete remote file
sftp> bye                                     # Disconnect
```

### Batch Mode (Non-Interactive)

```bash
# Execute commands from a batch file
sftp -b commands.txt user@host

# commands.txt:
# cd /inbound
# mget *.csv
# bye

# Single-command download
sftp user@host:/inbound/file.csv /data/landing/
```

### scp — Secure Copy

```bash
# Download file
scp user@host:/remote/path/file.csv /local/path/

# Upload file
scp /local/path/file.csv user@host:/remote/path/

# Recursive directory copy
scp -r user@host:/remote/directory/ /local/path/

# With specific key and port
scp -i ~/.ssh/id_pipeline -P 2222 file.csv user@host:/inbound/
```

### rsync over SSH

```bash
# Sync remote directory to local (incremental, compressed)
rsync -avz -e "ssh -i ~/.ssh/id_pipeline" \
    user@host:/remote/data/ /local/landing/

# Sync with delete (mirror)
rsync -avz --delete -e ssh user@host:/source/ /destination/

# Dry run first
rsync -avzn -e ssh user@host:/source/ /destination/

# Bandwidth limit (KB/s)
rsync -avz --bwlimit=5000 -e ssh user@host:/source/ /destination/
```

---

## 5. Python SFTP with Paramiko

### Installation

```bash
pip install paramiko
```

### Connection and Basic Operations

```python
import paramiko
from pathlib import Path


def create_sftp_client(
    host: str,
    username: str,
    key_path: str | None = None,
    password: str | None = None,
    port: int = 22,
) -> tuple[paramiko.SSHClient, paramiko.SFTPClient]:
    """Create an SFTP client with key-based or password authentication."""
    ssh = paramiko.SSHClient()
    ssh.set_missing_host_key_policy(paramiko.RejectPolicy())  # Never auto-accept
    ssh.load_system_host_keys()
    ssh.load_host_keys(str(Path.home() / ".ssh" / "known_hosts"))

    connect_kwargs = {"hostname": host, "port": port, "username": username}
    if key_path:
        connect_kwargs["key_filename"] = key_path
    elif password:
        connect_kwargs["password"] = password

    ssh.connect(**connect_kwargs)
    sftp = ssh.open_sftp()
    return ssh, sftp
```

### Upload and Download

```python
def upload_file(sftp: paramiko.SFTPClient, local_path: str, remote_path: str) -> None:
    """Upload a file with progress callback."""
    file_size = Path(local_path).stat().st_size

    def progress(transferred: int, total: int) -> None:
        pct = (transferred / total) * 100
        print(f"\rUploading: {pct:.1f}%", end="", flush=True)

    sftp.put(local_path, remote_path, callback=progress)
    print(f"\nUploaded {local_path} -> {remote_path} ({file_size:,} bytes)")


def download_file(sftp: paramiko.SFTPClient, remote_path: str, local_path: str) -> None:
    """Download a file from SFTP."""
    sftp.get(remote_path, local_path)
    print(f"Downloaded {remote_path} -> {local_path}")
```

### Directory Listing and Filtering

```python
import stat
from datetime import datetime


def list_remote_files(
    sftp: paramiko.SFTPClient,
    remote_dir: str,
    pattern: str = "*",
) -> list[dict]:
    """List files in a remote directory with metadata."""
    from fnmatch import fnmatch

    results = []
    for entry in sftp.listdir_attr(remote_dir):
        if stat.S_ISREG(entry.st_mode) and fnmatch(entry.filename, pattern):
            results.append({
                "filename": entry.filename,
                "size_bytes": entry.st_size,
                "modified": datetime.fromtimestamp(entry.st_mtime),
                "path": f"{remote_dir}/{entry.filename}",
            })
    return sorted(results, key=lambda x: x["modified"])
```

### Context Manager Pattern

```python
from contextlib import contextmanager


@contextmanager
def sftp_connection(host: str, username: str, key_path: str, port: int = 22):
    """Context manager for safe SFTP connection handling."""
    ssh, sftp = create_sftp_client(host, username, key_path=key_path, port=port)
    try:
        yield sftp
    finally:
        sftp.close()
        ssh.close()


# Usage
with sftp_connection("sftp.vendor.com", "svc_ingest", "/keys/id_vendor") as sftp:
    files = list_remote_files(sftp, "/inbound", "*.csv")
    for f in files:
        download_file(sftp, f["path"], f"/data/landing/{f['filename']}")
```

---

## 6. Automated File Transfer Patterns

### Pattern 1: Polling Directory

```python
import time
import logging

logger = logging.getLogger(__name__)


def poll_and_ingest(
    sftp_config: dict,
    remote_dir: str,
    local_dir: str,
    pattern: str = "*.csv",
    poll_interval_seconds: int = 300,
    archive_dir: str = "/archive",
) -> None:
    """Poll a remote SFTP directory on a schedule, download new files."""
    seen_files: set[str] = set()

    while True:
        try:
            with sftp_connection(**sftp_config) as sftp:
                files = list_remote_files(sftp, remote_dir, pattern)
                new_files = [f for f in files if f["filename"] not in seen_files]

                for f in new_files:
                    local_path = f"{local_dir}/{f['filename']}"
                    download_file(sftp, f["path"], local_path)
                    # Move to archive on remote
                    sftp.rename(f["path"], f"{archive_dir}/{f['filename']}")
                    seen_files.add(f["filename"])
                    logger.info(f"Ingested {f['filename']}")

        except Exception as e:
            logger.error(f"SFTP poll failed: {e}")

        time.sleep(poll_interval_seconds)
```

### Pattern 2: Event-Driven (Airflow Sensor)

```python
# Airflow SFTP sensor — waits for file to appear, then triggers downstream
from airflow.providers.sftp.sensors.sftp import SFTPSensor
from airflow.providers.sftp.operators.sftp import SFTPOperator

sftp_sensor = SFTPSensor(
    task_id="wait_for_vendor_file",
    sftp_conn_id="vendor_sftp",
    path="/inbound/daily_feed_{{ ds_nodash }}.csv",
    poke_interval=120,
    timeout=3600,
    mode="reschedule",  # Free up worker slot between checks
)

sftp_download = SFTPOperator(
    task_id="download_vendor_file",
    sftp_conn_id="vendor_sftp",
    remote_filepath="/inbound/daily_feed_{{ ds_nodash }}.csv",
    local_filepath="/data/landing/daily_feed_{{ ds_nodash }}.csv",
    operation="get",
)
```

### Pattern 3: Scheduled with Marker Files

```python
def transfer_with_marker(sftp, remote_dir: str, local_dir: str) -> bool:
    """Only download data files when a .done marker file exists."""
    files = [f.filename for f in sftp.listdir_attr(remote_dir)]
    marker_files = [f for f in files if f.endswith(".done")]

    if not marker_files:
        logger.info("No marker files found — skipping")
        return False

    for marker in marker_files:
        data_file = marker.replace(".done", "")
        if data_file in files:
            sftp.get(f"{remote_dir}/{data_file}", f"{local_dir}/{data_file}")
            sftp.remove(f"{remote_dir}/{marker}")
            logger.info(f"Transferred {data_file} (marker: {marker})")

    return True
```

---

## 7. GoAnywhere MFT and Managed File Transfer

### When to Use Enterprise MFT vs Scripted SFTP

| Factor | Scripted SFTP | Managed File Transfer (MFT) |
|---|---|---|
| **Number of partners** | Few (< 10) | Many (10+), diverse protocols |
| **Compliance requirements** | Basic logging sufficient | PCI-DSS, HIPAA, SOX audit trails |
| **Protocols needed** | SFTP only | SFTP + FTPS + AS2 + HTTP + MQ |
| **Team skills** | Engineers comfortable with scripts | Mixed technical levels |
| **Monitoring** | Custom dashboards / alerting | Built-in dashboards, SLA tracking |
| **File transformations** | Handled downstream | Inline PGP encryption, format conversion |
| **Cost sensitivity** | Low operational cost | Licence cost justified by scale |
| **Key/credential management** | Manual or via secrets manager | Centralised within MFT platform |

### GoAnywhere MFT Overview

GoAnywhere MFT is an enterprise managed file transfer platform by Fortra (formerly HelpSystems). Key capabilities:

- **Multi-protocol support** — SFTP, FTPS, SCP, HTTP/S, AS2, SMTP, cloud connectors
- **Project-based workflows** — XML-defined transfer jobs with steps, conditions, and error handling
- **Centralised key management** — SSH keys, PGP keys, and TLS certificates managed in one console
- **Audit and compliance** — immutable transfer logs, tamper detection, detailed reporting
- **Scheduling and triggers** — cron-style schedules, file monitors, event-driven execution
- **Clustering** — active-active for high availability

### GoAnywhere XML Configuration Example

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Project>
    <Name>IngestVendorFeed</Name>
    <Description>Download daily CSV from vendor SFTP to landing zone</Description>
    <Steps>
        <Step>
            <Type>SFTP</Type>
            <Action>download</Action>
            <Host>sftp.vendor.example.com</Host>
            <Port>22</Port>
            <Username>${vendor_sftp_user}</Username>
            <AuthenticationMethod>sshkey</AuthenticationMethod>
            <RemoteDirectory>/outbound/daily/</RemoteDirectory>
            <LocalDirectory>/data/landing/vendor/</LocalDirectory>
            <FilePattern>feed_*.csv</FilePattern>
            <Overwrite>no</Overwrite>
            <Timeout>600</Timeout>
            <Retries>3</Retries>
        </Step>
        <Step>
            <Type>Notification</Type>
            <Subject>Vendor Feed Ingested</Subject>
            <Body>Daily feed downloaded to landing zone.</Body>
            <RecipientEmail>data-team@company.com</RecipientEmail>
        </Step>
    </Steps>
    <PostFailureAction>
        <Type>LogMessage</Type>
        <Message>Vendor feed download failed — check SFTP connectivity.</Message>
    </PostFailureAction>
</Project>
```

### Alternative MFT Platforms

| Platform | Notes |
|---|---|
| **GoAnywhere MFT** | Strong workflow engine, good Mainframe support |
| **IBM Sterling File Gateway** | Enterprise-scale, AS2 heavy |
| **Axway SecureTransport** | API-first, cloud-native options |
| **Globalscape EFT** | Windows-centric, good AD integration |
| **AWS Transfer Family** | Managed SFTP/FTPS backed by S3 or EFS |
| **Azure SFTP for Blob Storage** | Native Azure Blob SFTP endpoint |

---

## 8. Security Best Practices

### Server Hardening

```bash
# /etc/ssh/sshd_config — hardened for SFTP-only access
Protocol 2
PermitRootLogin no
PasswordAuthentication no              # Key-based only
PubkeyAuthentication yes
MaxAuthTries 3
LoginGraceTime 30

# Restrict SFTP users to chroot
Match Group sftp_users
    ChrootDirectory /data/sftp/%u
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
    PermitTunnel no
```

### Chroot Configuration

```bash
# Directory structure for chrooted SFTP user
sudo mkdir -p /data/sftp/vendor1/{inbound,outbound,archive}
sudo chown root:root /data/sftp/vendor1
sudo chmod 755 /data/sftp/vendor1
sudo chown vendor1:sftp_users /data/sftp/vendor1/{inbound,outbound,archive}
```

> [!important] The chroot directory and every parent directory must be owned by root and not writable by any other user. Only subdirectories within the chroot can be owned by the SFTP user.

### IP Whitelisting

```bash
# /etc/ssh/sshd_config — restrict by source IP
Match User vendor1
    AllowUsers vendor1@203.0.113.10
    AllowUsers vendor1@203.0.113.11

# Or via firewall (iptables)
iptables -A INPUT -p tcp --dport 22 -s 203.0.113.10/32 -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j DROP
```

### Key Rotation Procedure

1. **Generate new key pair** on the client side
2. **Add new public key** to server `authorized_keys` (keep old key active)
3. **Test connectivity** with the new key
4. **Update all pipelines** to reference the new private key
5. **Remove old public key** from `authorized_keys` after confirmation
6. **Archive old key pair** securely with rotation date recorded

### Audit Logging

```bash
# Enhanced SFTP logging in sshd_config
Subsystem sftp internal-sftp -l INFO -f AUTH

# Log all file operations
Subsystem sftp internal-sftp -l VERBOSE
```

Centralise SFTP logs to your SIEM or log aggregation platform. Key events to monitor:

- Failed authentication attempts
- Connections from unexpected IP addresses
- Unusual file sizes or transfer volumes
- Transfers outside expected time windows

---

## 9. Data Engineering Context

### Landing Zone Pattern

```
/data/
├── landing/              # Raw files from SFTP
│   ├── vendor_a/
│   │   ├── orders_20260315.csv
│   │   └── orders_20260314.csv
│   └── vendor_b/
│       └── inventory_20260315.parquet
├── staging/              # Validated, schema-checked
│   ├── vendor_a/
│   └── vendor_b/
├── archive/              # Processed files (retained for replay)
│   ├── vendor_a/
│   │   └── 2026/03/
│   └── vendor_b/
│       └── 2026/03/
└── rejected/             # Files that failed validation
    └── vendor_a/
```

### File-Based Ingestion Pipeline

```
SFTP Server ──> Landing Zone ──> Validation ──> Stage to Warehouse ──> Archive
     │                │               │                │                  │
     │          Download file    Check schema     COPY INTO /         Move to
     │          + verify hash    + row counts     MERGE               archive/
     │                           + nulls                              + log
```

### Archive-After-Load Pattern

```python
from datetime import datetime
from pathlib import Path
import shutil


def archive_after_load(
    landing_dir: str,
    archive_base: str,
    filename: str,
) -> str:
    """Move processed file to date-partitioned archive."""
    today = datetime.now()
    archive_dir = Path(archive_base) / f"{today:%Y}" / f"{today:%m}" / f"{today:%d}"
    archive_dir.mkdir(parents=True, exist_ok=True)

    source = Path(landing_dir) / filename
    destination = archive_dir / filename

    shutil.move(str(source), str(destination))
    return str(destination)
```

### Integration with [[Snowflake]] Stages

```sql
-- External stage pointing at landing zone (S3 example)
CREATE OR REPLACE STAGE raw.vendor_a_stage
    URL = 's3://data-lake/landing/vendor_a/'
    STORAGE_INTEGRATION = s3_integration
    FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1 FIELD_OPTIONALLY_ENCLOSED_BY = '"');

-- Load from stage after SFTP delivery
COPY INTO raw.orders
    FROM @raw.vendor_a_stage
    PATTERN = 'orders_.*\\.csv'
    ON_ERROR = 'SKIP_FILE';
```

### Integration with [[Airflow]]

```python
# Full DAG: sense file -> download -> validate -> load -> archive
from airflow import DAG
from airflow.providers.sftp.sensors.sftp import SFTPSensor
from airflow.operators.python import PythonOperator

with DAG("sftp_ingest_vendor_a", schedule="0 6 * * *", catchup=False) as dag:
    sense = SFTPSensor(
        task_id="sense_file",
        sftp_conn_id="vendor_a_sftp",
        path="/outbound/orders_{{ ds_nodash }}.csv",
        mode="reschedule",
        poke_interval=300,
        timeout=7200,
    )
    download = PythonOperator(task_id="download", python_callable=download_from_sftp)
    validate = PythonOperator(task_id="validate", python_callable=validate_file)
    load = PythonOperator(task_id="load_to_warehouse", python_callable=copy_into_snowflake)
    archive = PythonOperator(task_id="archive", python_callable=archive_after_load)

    sense >> download >> validate >> load >> archive
```

---

## Related Notes

- [[SSH]]
- [[REST API]]
- [[Airflow Deep Dive]]
- [[Snowflake Data Loading]]
- [[Docker Networking]]
- [[Terraform AWS Modules]]
- [[Data Quality and Testing]]
- [[ETL Pipeline Templates]]
