# BloodHound Loot Collection

A practical guide to launching **BloodHound Community Edition (CE)** and collecting Active Directory data with five tools.

Use these commands in your own lab or an environment you are authorized to assess. “Loot” here means directory and relationship data—not dumped passwords. **Collection** produces the files; **ingestion** imports them into BloodHound for analysis.

## Choose a collector

| Starting point | Collector | Check before collecting |
| --- | --- | --- |
| Authorized Windows session | SharpHound CE | Version compatible with your BloodHound deployment |
| Linux with domain credentials | NetExec or BloodHound.py CE | AD DNS resolution and installed CE executable |
| Cross-platform collection | RustHound CE | output path and DNS versus LDAP target flags |
| Basic directory collection | bloodyAD | Confirm coverage for the object and relationship types you need |

Run one collector first and verify the resulting upload before combining datasets. Keep the run log so you can trace missing edges to permissions or reachability.

## Contents

- [Launch BloodHound](#launch-bloodhound)
- [Prepare your lab](#prepare-your-lab)
- [1. NetExec](#1-netexec)
- [2. SharpHound](#2-sharphound)
- [3. RustHound CE](#3-rusthound-ce)
- [4. BloodHound.py](#4-bloodhoundpy)
- [5. bloodyAD](#5-bloodyad)
- [Import and verify](#import-and-verify)
- [Troubleshooting](#troubleshooting)

## Launch BloodHound

The BloodHound CLI manages the container deployment through Docker Compose. Install Docker with Compose v2 first and make sure the daemon is running. The official quickstart specifies at least 8 GB RAM, 4 CPU cores, and 10 GB disk space.

### 1. Download

For an x86-64 Kali guest, choose Linux AMD64 regardless of the host OS. ARM guests require an ARM build.

[Latest releases](https://github.com/SpecterOps/bloodhound-cli/releases) · [Original v0.2.1 reference](https://github.com/SpecterOps/bloodhound-cli/releases/tag/v0.2.1)

```bash
wget https://github.com/SpecterOps/bloodhound-cli/releases/latest/download/bloodhound-cli-linux-amd64.tar.gz
```

### 2. Extract and add to PATH

```bash
tar -xf bloodhound-cli-linux-amd64.tar.gz
chmod +x bloodhound-cli
sudo mv bloodhound-cli /usr/local/bin/
```

### 3. Install

```bash
sudo bloodhound-cli install
```

Wait for startup to finish and save the generated administrator password. These examples consistently use `sudo`; if Docker works without it, run the CLI consistently as your regular user instead.

### 4. Recover the initial password

```bash
sudo bloodhound-cli config get default_password
```

This retrieves the stored initial value where supported; it does not reveal a subsequently changed password. The current quickstart documents a reset:

```bash
sudo bloodhound-cli resetpwd
```

### 5. Log in

Open <http://127.0.0.1:8080/ui/login> on the machine running BloodHound. Sign in as `admin` and change the password when prompted. In a VM, localhost refers to that VM.

Source: [SpecterOps CE quickstart](https://bloodhound.specterops.io/get-started/quickstart/community-edition-quickstart).

## Prepare your lab

Replace every bracketed placeholder before running a command. Install the chosen collector from its linked upstream project and check its local `--help` output.

| Placeholder | Meaning | Example only |
| --- | --- | --- |
| `[DC-IP]` | Domain controller IP | `192.168.56.10` |
| `[DC-HOSTNAME]` | DC fully qualified DNS name | `dc01.lab.example` |
| `[DOMAIN]` | AD DNS domain | `lab.example` |
| `username` | Authorized domain account | `pentest` |
| `password` | That account's password | Your lab password |

Use the credentials assigned to your own training environment:

```text
Username: <LAB_USERNAME>
Password: <LAB_PASSWORD>
```

These placeholders are not BloodHound defaults. Keep real credentials and collected data out of GitHub. Passwords passed as command arguments may appear in shell history and process listings.

Resolve the AD domain through its DNS server and ensure the lab network is reachable. `All` is broad collection, not a guarantee of complete results: permissions, unavailable hosts, and collector coverage affect the graph. Some collection methods contact member computers over SMB/RPC in addition to querying LDAP.

## 1. NetExec

[Project documentation](https://www.netexec.wiki/ldap-protocol/bloodhound-ingestor)

Use this from Linux when you already have domain credentials and want collection integrated into NetExec.

```bash
nxc ldap [DC-IP] -u 'username' -p 'password' --bloodhound --collection All --dns-server [DC-IP]
```

| Argument | Purpose |
| --- | --- |
| `ldap [DC-IP]` | Select the LDAP protocol and DC target |
| `-u`, `-p` | Authenticate with the domain account |
| `--bloodhound` | Enable BloodHound collection |
| `--collection All` | Request the collector's broad collection set |
| `--dns-server` | Use AD DNS for name resolution |

Current NetExec documentation identifies CE as the default ingestor. Check your configuration if using an older installation. Read the completion output for the generated archive location; do not assume it is in the current folder.

## 2. SharpHound

[Official releases](https://github.com/SpecterOps/SharpHound/releases) · [Flag reference](https://bloodhound.specterops.io/collect-data/ce-collection/sharphound-flags)

SharpHound is SpecterOps' Windows collector. Run the CE-compatible executable in an authorized domain user context on a machine that can reach the domain.

```powershell
.\SharpHound.exe -c All
```

`-c` selects collection methods. `All` requests the broad set supported by that release. The current Windows identity supplies the authentication context; copying the executable onto a machine does not grant domain access.

To choose an output directory:

```powershell
.\SharpHound.exe -c All --OutputDirectory C:\Temp\BloodHound
```

Transfer the resulting ZIP to the machine with access to your BloodHound UI. A privileged session is not required for every directory query, but some host-level collection needs additional rights. Older tutorials may mention `SharpHound.ps1`; do not assume it is bundled with a current release.

## 3. RustHound CE

[Releases](https://github.com/g0h4n/RustHound-CE/releases) · [Usage reference](https://github.com/g0h4n/RustHound-CE/blob/main/HELP.md)

A cross-platform Rust collector for BloodHound CE that does not require .NET. Use the CE build, whose documented executable name is `rusthound-ce`.

```bash
rusthound-ce -d [DOMAIN] -u 'username' -p 'password' -i <IP> -c All
```

| Argument | Purpose |
| --- | --- |
| `-d` | AD DNS domain |
| `-u`, `-p` | LDAP identity and password; UPN shown here |
| `i` | Ip address 
|`c` | Collection methods 

Add `-i [DC-IP]` if you need to specify the LDAP server IP explicitly. The supplied `rusthound` command uses an older/different executable name; verify the installed binary before copying it. Output goes into `./rusthound_output`.

## 4. BloodHound.py

[CE branch and installation instructions](https://github.com/dirkjanm/BloodHound.py/tree/bloodhound-ce)

A Python collector for remote collection from Linux. The CE package installs `bloodhound-ce-python`; the legacy package installs `bloodhound-python`.

```bash
bloodhound-ce-python -u 'username' -p 'password' -d [DOMAIN] -dc [DC-HOSTNAME] -c All -ns [DC-IP] --zip
```

| Argument | Purpose |
| --- | --- |
| `-u`, `-p` | Domain credentials |
| `-d` | AD DNS domain |
| `-dc` | DC hostname, rather than its IP |
| `-c All` | Broad collection selection |
| `-ns` | DNS server IP |
| `--zip` | Package output into a ZIP |

Run it from a dedicated output directory. The originally supplied `bloodhound-python` syntax belongs to the legacy executable; use the CE command above for this guide. Installing the wrong package can produce an incompatible dataset even when collection succeeds.

## 5. bloodyAD

[Project](https://github.com/CravateRouge/bloodyAD) · [Collector reference](https://github.com/CravateRouge/bloodyAD/wiki/User-Guide#get-bloodhound)

bloodyAD includes a basic CE collector. It can provide an alternative when another collector fails, but its coverage differs.

```bash
bloodyAD -H [DC-HOSTNAME] -d [DOMAIN] -u 'username' -p 'password' get bloodhound
```

| Argument | Purpose |
| --- | --- |
| `-H` | DC hostname or IP |
| `-d` | Authentication domain |
| `-u`, `-p` | Domain credentials |
| `get bloodhound` | Collect and export BloodHound CE data |

The documented default writes the ZIP to the current path; `get bloodhound --help` describes `--path` for choosing its destination. On case-sensitive systems, use the installed executable's exact name, normally `bloodyAD`.

The upstream guide labels this collector as under development, with basics supported and complex nodes such as ADCS ESC coverage absent. Do not treat it as equivalent to SharpHound. A lab observation that it worked on Windows Server 2025 does not establish that other collectors universally fail there; versions and authentication policy matter.

## Import and verify

1. Log in to BloodHound CE.
2. Open **Quick Upload** and choose the collector's ZIP or supported JSON files.
3. Upload and wait for ingestion and analysis to complete.
4. Check for errors and confirm your expected domain, users, groups, and computers appear.
5. Compare the graph with collection logs. Missing session edges can reflect collection permissions or timing rather than an absence of sessions.

Record the collector version, collection time, selected methods, and failed hosts alongside each dataset. Keep results from separate labs in separate directories.

## Troubleshooting

| Symptom | First checks |
| --- | --- |
| Docker/Compose failure | Daemon running, Compose v2 available, sufficient resources, consistent CLI user |
| UI unavailable | Startup completion, port 8080, and whether your browser is on the VM or host |
| DNS or DC discovery failure | AD DNS server, DC FQDN, VPN route, and domain spelling |
| Authentication failure | Account/domain format, password quoting, account state, and clock synchronization |
| LDAP signing or TLS errors | Collector version and supported authentication settings; inspect the actual error |
| Partial host collection | Endpoint reachability, SMB/RPC access, and account permissions |
| Import rejected | CE versus legacy format, archive contents, collector version, and ingest error details |
| Unknown command or flag | Installed executable name and that version's help output |

## About this guide

Based on the supplied training exercise, rewritten with explanations and upstream references. Course screenshots remain with their original publisher. Commands were checked against documentation; no live AD collection was executed to validate this guide.

## Usage terms

Original material owned by 0xjyothendra requires written permission for modification or redistribution. Third-party content, assets, and earlier licensed releases retain their applicable terms. See [LICENSE](LICENSE).
