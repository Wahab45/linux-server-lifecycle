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

![Systeminfo: Windows, RAM, CPU und freier Speicher](screenshots/01-systeminfo.png)

- Hyper-V geprüft (als Administrator):

```powershell
Get-VM
Get-VMSwitch
Get-VMHost | Select-Object VirtualMachinePath, VirtualHardDiskPath
```

![Get-VM und Get-VMSwitch als Administrator](screenshots/03-get-vm-get-vmswitch-admin.png)

- Vorhandene VM `Ubuntu-Server-Base` genauer angeschaut:

```powershell
Get-VM -Name Ubuntu-Server-Base | Format-List Name, Generation, ProcessorCount, MemoryStartup, DynamicMemoryEnabled, Path
Get-VMHardDiskDrive -VMName Ubuntu-Server-Base | ForEach-Object { Get-VHD -Path $_.Path } | Select-Object Path, VhdType, Size, FileSize
Get-VMNetworkAdapter -VMName Ubuntu-Server-Base | Select-Object Name, SwitchName
Get-VMSnapshot -VMName Ubuntu-Server-Base
```

![Details der Basis-VM und ihres virtuellen Datenträgers](screenshots/04-vm-details.png)

- Netzwerk geprüft:

```powershell
Get-NetNat
Get-NetIPAddress -InterfaceAlias "vEthernet (LabSwitch-Internal)" -AddressFamily IPv4 | Select-Object IPAddress, PrefixLength
```

![Netzwerk: Adapter der VM, NAT und Host-Adresse](screenshots/05-netzwerk.png)

- Ergebnis: Generation-2-VM mit 3 GB Startspeicher (dynamisch). Die Festplatte ist dynamisch mit maximal 40 GB (ca. 13,8 GB belegt). Es gibt keine Checkpoints. Die VM hat zwei Netzwerkkarten: `LabSwitch-Internal` (Lab-Netz, Host-Adresse 192.168.100.1/24) und `Default Switch` (Internet). Ein NAT ist nicht eingerichtet.

**Probleme und Lösungen:**
- Problem: `Get-VM` und `Get-VMSwitch` zeigten den Fehler „Sie besitzen nicht die erforderliche Berechtigung für diese Aufgabe“.

![Fehlermeldung bei fehlenden Administrator-Rechten](screenshots/02-fehler-berechtigung.png)

  Lösung: PowerShell war nicht als Administrator gestartet. Mit Rechtsklick → „Als Administrator ausführen“ funktionierten beide Befehle.
- Annahme: Ich dachte zuerst, für Internet im Lab-Netz sei ein NAT nötig (`Get-NetNat` war leer).
  Lösung: Das ist nicht nötig. Die zweite Netzwerkkarte am `Default Switch` liefert das Internet. Das Lab-Netz bleibt getrennt und ist für die Verwaltung da.

**Was ich gelernt habe:**
- Hyper-V kennt drei Switch-Typen: External (echtes Netzwerk), Internal (nur Host und VMs), Private (nur VMs). Mit einem Internal-Switch ist das Lab vom echten Netzwerk getrennt.
- Fehler bei Hyper-V-Befehlen kommen oft von fehlenden Administrator-Rechten.
- Bytes in GB umrechnen: ÷ 1024³.
- Eine Basis-VM wird nicht direkt benutzt, sondern geklont. Beim Klonen von Linux-VMs müssen `machine-id` und SSH-Host-Keys neu erzeugt werden.
