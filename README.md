# linux-server-lifecycle

Lab-Projekt: Ubuntu-Server mit Ansible bereitstellen, härten, prüfen, sichern und wiederherstellen.

### Phase 0.1 — Projektplanung und Abgrenzung

**Durchgeführte Schritte:**
- Projektidee bewertet: Linux-Server bereitstellen, absichern, prüfen, sichern und wiederherstellen als **ein zusammenhängender Ablauf**.
- Umfang begrenzt: nur ein Ziel-Server, keine Monitoring-Lösung (SIEM), nur eine Auswahl von CIS-Kontrollen (Level 1), nur ein Backup-Werkzeug.
- Reihenfolge festgelegt: Grundlagen → Provisioning → Hardening & Audit → Backup & Restore → kompletter Durchlauf → Dokumentation.
- Architektur geplant:

```mermaid
flowchart LR
    A["Control Node (Ansible)"] -->|SSH| B["Ziel-Server (Ubuntu)"]
    B -->|Backup| C["Backup-Ziel"]
```

**Probleme und Lösungen:**
- Problem: „Bereitstellung“ kann zwei Dinge bedeuten: (1) die VM erstellen, (2) das System danach konfigurieren. Ansible macht vor allem (2).
  Lösung: Ich trenne beide Ebenen und entscheide später, wie weit ich automatisiere.
- Problem: Der ursprüngliche Umfang war zu groß.
  Lösung: Umfang reduziert (siehe oben). Der Fokus liegt auf einem getesteten Gesamtdurchlauf.

**Was ich gelernt habe:**
- Ansible arbeitet ohne Agent („agentless“): Es verbindet sich per SSH. Das Ziel braucht nur SSH, Python und einen Benutzer mit `sudo`.
- Control Node (Steuerrechner) und Managed Node (Ziel-Server) sind zwei getrennte Rollen. Auf Windows läuft Ansible nicht direkt als Control Node. Ich brauche WSL2 oder eine eigene Linux-VM.
- Idempotenz: Ein Playbook kann mehrmals laufen und liefert immer dasselbe Ergebnis.
- Ein Backup auf demselben Server ist kein echtes Backup.
- Ein Audit braucht einen Vorher-Nachher-Vergleich, sonst gibt es keinen Beweis.

### Phase 0.2 — Lab-Umgebung prüfen (Hyper-V)

**Durchgeführte Schritte:**
- Rechner geprüft (nur lesende Befehle):

```powershell
Get-ComputerInfo | Select-Object WindowsProductName, WindowsVersion, OsTotalVisibleMemorySize
Get-CimInstance Win32_Processor | Select-Object Name, NumberOfCores, NumberOfLogicalProcessors
Get-PSDrive -PSProvider FileSystem
```

- Ergebnis: Windows 10 Pro, ca. 13,9 GB RAM (14.528.572 KB ÷ 1024 ÷ 1024), Ryzen 7 mit 8 Kernen / 16 Threads, ca. 330 GB freier Speicher. Das reicht für drei kleine VMs.

![Systeminfo: Windows, RAM, CPU und freier Speicher](Screenshots/01-systeminfo.png)

- Hyper-V geprüft (als Administrator):

```powershell
Get-VM
Get-VMSwitch
Get-VMHost | Select-Object VirtualMachinePath, VirtualHardDiskPath
```

![Get-VM und Get-VMSwitch als Administrator](Screenshots/03-get-vm-get-vmswitch-admin.png)

- Vorhandene VM `Ubuntu-Server-Base` genauer angeschaut:

```powershell
Get-VM -Name Ubuntu-Server-Base | Format-List Name, Generation, ProcessorCount, MemoryStartup, DynamicMemoryEnabled, Path
Get-VMHardDiskDrive -VMName Ubuntu-Server-Base | ForEach-Object { Get-VHD -Path $_.Path } | Select-Object Path, VhdType, Size, FileSize
Get-VMNetworkAdapter -VMName Ubuntu-Server-Base | Select-Object Name, SwitchName
Get-VMSnapshot -VMName Ubuntu-Server-Base
```

![Details der Basis-VM und ihres virtuellen Datenträgers](Screenshots/04-vm-details.png)

- Netzwerk geprüft:

```powershell
Get-NetNat
Get-NetIPAddress -InterfaceAlias "vEthernet (LabSwitch-Internal)" -AddressFamily IPv4 | Select-Object IPAddress, PrefixLength
```

![Netzwerk: Adapter der VM, NAT und Host-Adresse](Screenshots/05-netzwerk.png)

- Ergebnis: Generation-2-VM mit 3 GB Startspeicher (dynamisch). Die Festplatte ist dynamisch mit maximal 40 GB (ca. 13,8 GB belegt). Es gibt keine Checkpoints. Die VM hat zwei Netzwerkkarten: `LabSwitch-Internal` (Lab-Netz, Host-Adresse 192.168.100.1/24) und `Default Switch` (Internet). Ein NAT ist nicht eingerichtet.

**Probleme und Lösungen:**
- Problem: `Get-VM` und `Get-VMSwitch` zeigten den Fehler „Sie besitzen nicht die erforderliche Berechtigung für diese Aufgabe“.

![Fehlermeldung bei fehlenden Administrator-Rechten](Screenshots/02-fehler-berechtigung.png)

  Lösung: PowerShell war nicht als Administrator gestartet. Mit Rechtsklick → „Als Administrator ausführen“ funktionierten beide Befehle.
- Annahme: Ich dachte zuerst, für Internet im Lab-Netz sei ein NAT nötig (`Get-NetNat` war leer).
  Lösung: Das ist nicht nötig. Die zweite Netzwerkkarte am `Default Switch` liefert das Internet. Das Lab-Netz bleibt getrennt und ist für die Verwaltung da.

**Was ich gelernt habe:**
- Hyper-V kennt drei Switch-Typen: External (echtes Netzwerk), Internal (nur Host und VMs), Private (nur VMs). Mit einem Internal-Switch ist das Lab vom echten Netzwerk getrennt.
- Fehler bei Hyper-V-Befehlen kommen oft von fehlenden Administrator-Rechten.
- Bytes in GB umrechnen: ÷ 1024³.
- Eine Basis-VM wird nicht direkt benutzt, sondern geklont. Beim Klonen von Linux-VMs müssen `machine-id` und SSH-Host-Keys neu erzeugt werden.

### Phase 0.3 — VM erstellen und Ubuntu Server installieren

**Durchgeführte Schritte:**
- Neue VM für dieses Projekt erstellt (Generation 2, 2 GB RAM, 2 vCPU, 25 GB dynamische Festplatte).
- Secure Boot bleibt an, mit der Vorlage für Linux (sonst startet Ubuntu nicht).
- Die ISO-Datei (Ubuntu Server 24.04.4) hatte ich schon früher geprüft.

```powershell
$vmName = "lab-ubuntu2404-base"
$vmPath = "C:\HyperV-Labs\Lifecycle"
New-Item -ItemType Directory -Path "$vmPath\$vmName" -Force

New-VM -Name $vmName -Generation 2 -MemoryStartupBytes 2GB -Path $vmPath `
  -NewVHDPath "$vmPath\$vmName\$vmName.vhdx" -NewVHDSizeBytes 25GB `
  -SwitchName "LabSwitch-Internal"
Set-VMProcessor -VMName $vmName -Count 2
Set-VMMemory -VMName $vmName -DynamicMemoryEnabled $false
Add-VMNetworkAdapter -VMName $vmName -SwitchName "Default Switch"

Add-VMDvdDrive -VMName $vmName -Path "C:\HyperV-Labs\ISOs\ubuntu-24.04.4-live-server-amd64.iso"
Set-VMFirmware -VMName $vmName -SecureBootTemplate "MicrosoftUEFICertificateAuthority"
Set-VMFirmware -VMName $vmName -FirstBootDevice (Get-VMDvdDrive -VMName $vmName)
```

![VM-Einstellungen nach dem Erstellen](Screenshots/06-vm-erstellt.png)

- Installer-Auswahl: Sprache English, Ubuntu Server (nicht minimized), ganze Festplatte mit LVM (ohne Verschlüsselung), OpenSSH-Server installiert, keine Snaps.

![Installer: SSH-Konfiguration](Screenshots/07-installer-ssh.png)

- Nach der Installation die ISO getrennt:

```powershell
Set-VMDvdDrive -VMName $vmName -Path $null
```

![Erster Login](Screenshots/08-erster-login.png)

- Erste Prüfung: Ubuntu 24.04.4 LTS, Kernel 6.8.0-142-generic, Python 3.12.3. Die Festplatte hat 25 GB (LVM-Gruppe 21,95 GB, davon ca. 11 GB dem Root-Dateisystem zugeteilt und ca. 11 GB frei). 54 Updates sind verfügbar.

```bash
uname -r
lsblk
df -h /
sudo vgs
sudo lvs
ip -br a
ip -br link
systemctl is-active ssh
python3 --version
hostnamectl
```

![Erste Prüfung: Kernel, Festplatte, LVM, Netzwerk](Screenshots/09-erste-pruefung.png)

```bash
cat /etc/os-release
```

![Betriebssystem-Version](Screenshots/10-os-release.png)

**Probleme und Lösungen:**
- Problem: Am Ende der Installation kam „Failed unmounting cdrom.mount“.
  Lösung: ISO in Windows trennen (`Set-VMDvdDrive -VMName $vmName -Path $null`), danach in der VM Enter drücken.
- Problem: `systemctl is-active ssh` zeigte „inactive“.
  Lösung: Ubuntu 24.04 startet SSH per Socket Activation. Ich habe `systemctl is-active ssh.socket` (active) und `sudo ss -tlnp | grep ':22'` geprüft. Nach der ersten Verbindung war `ssh` active.

```bash
systemctl is-active ssh.socket
sudo ss -tlnp | grep ':22'
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
systemctl is-active ssh
```

![SSH-Socket, Port 22 und Fingerabdruck](Screenshots/11-ssh-socket.png)

- Problem: „Host key verification failed“ beim ersten `ssh`.
  Lösung: Ich hatte `y` und `n` getippt. Die Frage nach dem Fingerabdruck muss mit dem ganzen Wort `yes` beantwortet werden.

![Fehler beim Fingerabdruck](Screenshots/12-ssh-fingerprint-fehler.png)

- Problem: `Get-VMNetworkAdapter` zeigte „command not found“.
  Lösung: Ich war noch in der SSH-Sitzung (Linux). Mit `exit` beenden und den Befehl in Windows-PowerShell ausführen.

**Was ich gelernt habe:**
- Eine VM mit PowerShell zu erstellen ist wiederholbar und dokumentiert jede Einstellung.
- Generation-2-VMs brauchen für Linux die Secure-Boot-Vorlage „MicrosoftUEFICertificateAuthority“.
- Der Installer lässt bei LVM absichtlich Platz frei. Diesen Platz kann ich später für getrennte Partitionen und Snapshots nutzen.
- Ein SSH-Dienst kann „inactive“ sein und trotzdem funktionieren (Socket Activation).
- Den Fingerabdruck des Servers kann man mit `ssh-keygen -lf` prüfen, bevor man `yes` antwortet.
- Ich muss immer wissen, in welcher Shell ich bin: `PS C:\...>` ist Windows, `wahab@wahab:~$` ist Linux.

### Phase 0.4 — Lab-Netzwerk mit fester Adresse

**Durchgeführte Schritte:**
- Ausgangslage: `eth0` (Lab-Netz) war DOWN ohne Adresse. `eth1` (Default Switch) hatte eine Adresse per DHCP. In `/etc/netplan` stand nur `50-cloud-init.yaml` mit `eth1` und `dhcp4: true`.
- Zuordnung der Adapter über die MAC-Adressen geprüft (`00155D016F03` = eth0, `00155D016F04` = eth1).
- Die DHCP-Adresse von `eth1` hat sich mehrmals geändert (`172.19.119.199`, `172.19.123.205`, `172.19.116.21`). Sie ist für die Verwaltung nicht geeignet.
- Für dieses Projekt einen eigenen Internal-Switch angelegt, damit es keine Adresskonflikte mit anderen Lab-Netzen gibt:

```powershell
Get-NetIPAddress -AddressFamily IPv4 | Select-Object InterfaceAlias, IPAddress, PrefixLength
New-VMSwitch -Name "Lifecycle-Internal" -SwitchType Internal
New-NetIPAddress -InterfaceAlias "vEthernet (Lifecycle-Internal)" -IPAddress 10.99.0.1 -PrefixLength 24

Get-VMNetworkAdapter -VMName lab-ubuntu2404-base |
  Where-Object MacAddress -eq '00155D016F03' |
  Connect-VMNetworkAdapter -SwitchName "Lifecycle-Internal"
```

- In der VM eine eigene netplan-Datei für `eth0` erstellt:

```bash
sudo tee /etc/netplan/60-lab-netz.yaml > /dev/null <<'EOF'
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: false
      addresses:
        - 10.99.0.10/24
EOF
sudo chmod 600 /etc/netplan/60-lab-netz.yaml
sudo netplan generate
sudo netplan try
```

- Ergebnis: Ping von Windows (4 von 4 Antworten, 0 % Verlust) und SSH-Login über `10.99.0.10` funktionieren. Die Adapter der VM hängen an `Lifecycle-Internal` (eth0) und `Default Switch` (eth1).

![Adapter am neuen Switch, Ping und SSH-Login](Screenshots/14-ping-und-ssh.png)

- In der VM hat `eth0` jetzt `10.99.0.10/24`. Die Standardroute läuft nur über `eth1`.

```bash
ip -br a
ip route
```

![Adressen und Routen in der VM](Screenshots/15-netzwerk-nachher.png)

**Probleme und Lösungen:**
- Problem: `eth0` blieb nach `netplan generate` DOWN.
  Lösung: `netplan generate` prüft nur die Syntax. Erst `netplan try` (mit Enter) wendet die Konfiguration an.
- Problem: Ich habe beide Adapter mit dem Namen „Netzwerkkarte“.
  Lösung: Den Adapter über die MAC-Adresse auswählen (`Where-Object MacAddress -eq ...`).
- Hinweis: Ein Ping von der VM zum Windows-Host kann an der Windows-Firewall scheitern. Deshalb teste ich von Windows zur VM.

**Was ich gelernt habe:**
- Ein Internal-Switch verbindet nur Host und VMs, nicht das Firmennetz.
- Bei zwei Netzwerkkarten bekommt nur eine ein Standard-Gateway (hier `eth1` für das Internet). Deshalb steht in der netplan-Datei kein `gateway`.
- `netplan try` ist sicherer als `netplan apply`, weil es nach 120 Sekunden automatisch zurückrollt.
- `50-cloud-init.yaml` gehört cloud-init. Eigene Einstellungen kommen in eine eigene Datei.
- Adapter identifiziert man am besten über die MAC-Adresse.
