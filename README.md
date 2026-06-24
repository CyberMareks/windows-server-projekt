# 🪟 Windows-Server-Projekt — LF10B

> Praxisprojekt im Rahmen der Umschulung zum **Fachinformatiker Systemintegration**.  
> Aufbau und Administration einer vollständigen Windows-Server-2022-Unternehmensinfrastruktur — inklusive Active Directory, Hochverfügbarkeit, PKI, VPN, RADIUS und erweiterter Server-Administration.

***

## 🧰 Erworbene Kompetenzen

| Bereich | Technologien & Konzepte |
|---|---|
| **Virtualisierung** | Hyper-V (Windows Server 2022 als Hypervisor), VM-Automatisierung per PowerShell |
| **Active Directory** | AD DS, DNS-Integration, OU-Struktur, Benutzer-/Gruppenverwaltung, Replikation |
| **Gruppenrichtlinien (GPO)** | Erstellung, Verknüpfung, Erzwingung, Vererbung, Starter-GPOs, `gpupdate /force` |
| **Mehrdomänen-Umgebung** | Untergeordnete Domänen, neue Gesamtstrukturen, Vertrauensstellungen (bidirektional/unidirektional, selektive Authentifizierung) |
| **Hochverfügbarkeit** | Windows Failover Clustering, Cluster Shared Volumes (CSV), Node Fairness |
| **Speicher** | iSCSI-Target & Initiator, Windows-Speicherpools, Datendeduplizierung, BitLocker |
| **PKI / Zertifizierungsstellen** | AD CS (Root CA, Subordinate CA), Zertifikatsvorlagen, Autoenrollment, CRL/OCSP |
| **Netzwerkdienste** | DNS (Zonen, Zonenübertragung, Stub-Zonen, bedingte Weiterleitungen), DHCP |
| **VPN & Routing** | RRAS (Routing and Remote Access Service), Site-to-Site-VPN, NAT |
| **RADIUS / 802.1X** | Network Policy Server (NPS), RADIUS-Authentifizierung, Netzwerkzugriffssteuerung |
| **Serveradministration** | Windows Admin Center, Server-Manager, Aufgabenplanung, Serverumgebung betriebsbereit halten |
| **PowerShell-Automatisierung** | Skripte für VM-Bereitstellung, AD-Verwaltung, Netzwerkkonfiguration, Cluster-Setup |

***

## 🗺️ Netzwerkarchitektur

```
GFN-RLAB (Hyper-V Host — Windows Server 2022)
│
│   Privates Netz: 192.168.1.0/24  (Domänennetz — gfnlab.test)
│   SAN-Netz:      172.16.1.0/24   (iSCSI / Cluster-Traffic)
│   Heartbeat-Netz: 10.10.10.0/24  (Cluster-Heartbeat)
│
├── DC               (192.168.1.200)  — Domänencontroller #1, DNS, Globaler Katalog
├── Server1          (192.168.1.1)    — replizierender DC, iSCSI-Target (SAN: 172.16.1.1)
├── Server2          (192.168.1.2)    — Member-Server, Failover-Clusterknoten (SAN: 172.16.1.2, HB: 10.10.10.2)
├── Server3          (192.168.1.3)    — Member-Server, Failover-Clusterknoten (SAN: 172.16.1.3, HB: 10.10.10.3)
│                                       LXC-äquivalent: sub.gfnlab.test-DC (untergeordnete Domäne)
├── W11              (192.168.1.4)    — Windows 11 Client, Domänenmitglied, Testclient
│
└── Failover-Cluster FC (192.168.1.150) — Virtuelle IP des Clusters
    └── Geclusterter Dateiserver FS (192.168.1.151)
```

### Mehrdomänen-Topologie (Gesamtstrukturübersicht)

```
Gesamtstruktur: gfnlab.test
├── gfnlab.test          (DC — Stammdomäne)
│   └── sub.gfnlab.test  (Server2 — untergeordnete Domäne)
│
└── Gesamtstruktur-Vertrauensstellung (bidirektional)
    │
    └── it.pro           (Server3 — separate Gesamtstruktur)
```

***

## 🖥️ Systeme im Überblick

| Hostname | IP-Adresse | Betriebssystem | Rolle |
|---|---|---|---|
| **DC** | 192.168.1.200 | Windows Server 2022 | Primärer DC, DNS, Globaler Katalog, CA |
| **Server1** | 192.168.1.1 | Windows Server 2022 | Replizierender DC, iSCSI-Target, RRAS/VPN |
| **Server2** | 192.168.1.2 | Windows Server 2022 | Failover-Clusterknoten, Dateiserver, sub.gfnlab.test-DC |
| **Server3** | 192.168.1.3 | Windows Server 2022 | Failover-Clusterknoten, NPS/RADIUS, it.pro-DC |
| **W11** | 192.168.1.4 | Windows 11 | Domänenclient, Testclient |

***

## ⚙️ Dienste & Technologien

### 🏢 Active Directory Domain Services (AD DS)
- Aufbau der Domäne `gfnlab.test` mit DC als primärem Domänencontroller
- Server1 als **replizierender Domänencontroller** inkl. Global-Katalog-Server-Konfiguration
- OU-Struktur: `Arbeit`, `Administratoren`, `Notebooks` — mit gezielter GPO-Verknüpfung
- Benutzer- und Gruppenverwaltung per GUI und **PowerShell** (`New-ADUser`, `Add-ADGroupMember`)
- Vollautomatisierte VM-Bereitstellung mit `autounattend.xml` und PowerShell-Skripten
- **Sysvol-Replikation** und DSRM-Wiederherstellung

### 🌐 DNS (Windows DNS-Server)
- Integrierter AD-DNS mit Forward- und Reverse-Lookupzonen
- **Zonenübertragung** zwischen Gesamtstrukturen (gfnlab.test ↔ it.pro)
- Bedingte Weiterleitungen und Stub-Zonen als Alternativen zur Zonenübertragung
- AD-Standorte und Dienste: Standorte München, Wien, Bern — standortbasierte Anmeldeverteilung
- Bridgeheadserver-Konfiguration und zeitgesteuerte Replikation zwischen Standorten

### 📋 Gruppenrichtlinien (GPO)
- Erstellung und Verknüpfung von GPOs auf Domänen- und OU-Ebene
- Vererbungssteuerung: Erzwingung (`Enforced`), Vererbung deaktivieren
- Starter-GPOs für Firewall-Portkonfiguration (Remote-GPO-Aktualisierung)
- GPO-Anwendungsbeispiele: Skriptausführung, Desktop-Einstellungen, Anmelderichtlinien
- Fernaktualisierung: `Invoke-GPUpdate`, `gpupdate /force`, `gpresult /r`

### 🌲 Mehrdomänen-Umgebung & Vertrauensstellungen
- Untergeordnete Domäne `sub.gfnlab.test` auf Server2 (GUI und PowerShell)
- Separate Gesamtstruktur `it.pro` auf Server3
- **Gesamtstruktur-Vertrauensstellung** (bidirektional, gesamtstrukturweite Authentifizierung)
- Selektive Authentifizierung mit `Allow to authenticate`-Berechtigung
- UPN-Suffixe, Namensuffixrouting zwischen Gesamtstrukturen

### 💾 Speicher, iSCSI & Datendeduplizierung
- **iSCSI-Target** auf Server1 (Rolle `iSCSI-Zielserver`, PowerShell-Automatisierung)
  - 3 virtuelle Datenträger: 1× 10 GB (Quorum), 2× 50 GB (Clusterdaten)
  - Zugriffsbeschränkung auf SAN-IP per `Set-IscsiTargetServerSetting`
- **Windows-Speicherpools** mit virtuellen Datenträgern
- **Datendeduplizierung** für Dateiserver-Volumes
- **BitLocker** Laufwerkverschlüsselung mit TPM und Wiederherstellungsschlüssel

### 🔁 Failover Clustering (Hochverfügbarkeit)
- **Windows Server Failover Cluster** (FC) auf Server2 + Server3
  - Cluster-IP: `192.168.1.150`, dedizierte SAN- und Heartbeat-Netzwerke
  - Quorum-Konfiguration mit Datenträgerzeugen (Disk Witness, 10 GB)
  - **Cluster Shared Volume (CSV)** mit CSVFS-Dateisystem
- **Geclusterter Dateiserver** FS (`192.168.1.151`)
  - Bevorzugter Knoten: Server3, Failover-Konfiguration: 2 Fehler in 8 Stunden
  - SMB-Freigabe `Clustershare`, Live-Migration während Kopiervorgängen getestet
- Clusterfähiges Aktualisieren (CAU), Node Fairness

### 🔐 PKI — Zertifizierungsstellen (AD CS)
- **Root CA** und **untergeordnete CA** (Subordinate CA) mit AD-Integration
- Zertifikatsvorlagen, automatische Zertifikatsverteilung (Autoenrollment)
- CRL (Zertifikatsperrliste) und OCSP-Responder
- Zertifikatsverwaltung über `certsrv.msc`, `certmgr.msc`
- Sicherheitseinstellungen für Zertifizierungsstellen

### 🌐 VPN & Routing (RRAS)
- **Routing and Remote Access Service** (RRAS) auf Windows Server
- Site-to-Site-VPN-Konfiguration
- NAT-Routing für Internetzugang aus dem privaten Netz

### 🔑 RADIUS / NPS (802.1X-Netzwerkzugriffsteuerung)
- **Network Policy Server (NPS)** als RADIUS-Server
- Netzwerkrichtlinien für authentifizierten Netzwerkzugang
- 802.1X-basierte Zugriffskontrolle für Netzwerkgeräte

### 🛠️ Serveradministration & Betrieb
- **Windows Admin Center** für zentrale Serververwaltung
- **Aufgabenplanung** (Task Scheduler) für automatisierte Skriptausführung
- Überwachung, Protokollierung und Serverumgebung betriebsbereit halten
- Erweiterte DNS- und DHCP-Themen (Superscopes, Split-Brain-DNS, DHCP-Failover)
- PowerShell-Automatisierung durchgehend eingesetzt (VM-Erstellung, Cluster, AD, Netzwerk)

***

## 🗂️ Projektstruktur

```
windows-server-projekt/
├── README.md
└── docs/
    └── screenshots/
```

***

## 📌 Über dieses Projekt

Dieses Projekt wurde im Rahmen des Lernfelds 10B der Umschulung zum **Fachinformatiker Systemintegration** (GFN) praktisch im Remotelab (Hyper-V-basiert) durchgeführt. Schwerpunkte lagen auf Enterprise-typischen Szenarien: Active Directory-Verwaltung in Mehrdomänenumgebungen, Hochverfügbarkeit mit Failover-Clustering, PKI-Infrastruktur sowie Netzwerkzugriffskontrolle mit RADIUS/NPS. Alle Konfigurationen wurden sowohl über die GUI als auch per PowerShell umgesetzt.
