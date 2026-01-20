# Egyetemi Intranet - Összefoglaló Dokumentáció (Szakdolgozat)

> **Cél:** Egységes, biztonságos és bővíthető **egyetemi intranet** kialakítása (órarend, jegyek, hiányzások, belső információk).  
> **Környezet:** Proxmox (virtualizáció), OPNsense (tűzfal/router), Windows Server (AD/DNS/IIS), SQL Server (adatbázis), ASP.NET (webalkalmazás).

---

## Tartalomjegyzék
- [1. Bevezetés és célkitűzések](#1-bevezetés-és-célkitűzések)
- [2. Rendszer-architektúra](#2-rendszer-architektúra)
- [3. Hálózati topológia és DNS](#3-hálózati-topológia-és-dns)
- [4. Telepítés lépésről lépésre](#4-telepítés-lépésről-lépésre)
  - [4.1 Proxmox VE](#41-proxmox-ve)
  - [4.2 OPNsense](#42-opnsense)
  - [4.3 Windows Server - AD/DNS/IIS](#43-windows-server--addnsiis)
  - [4.4 SQL Server](#44-sql-server)
  - [4.5 ASPNET webalkalmazás](#45-aspnet-webalkalmazás)
- [5. Adatbázis-séma (minta)](#5-adatbázis-séma-minta)
- [6. Jogosultságkezelés](#6-jogosultságkezelés)
- [7. Biztonsági megfontolások](#7-biztonsági-megfontolások)
- [8. Üzemeltetés: mentés, monitoring](#8-üzemeltetés-mentés-monitoring)
- [9. Tesztelési ellenőrzőlista](#9-tesztelési-ellenőrzőlista)
- [10. Hibakeresés](#10-hibakeresés)
- [11. Bővíthetőség és jövőbeli fejlesztések](#11-bővíthetőség-és-jövőbeli-fejlesztések)

---

## 1. Bevezetés és célkitűzések
A projekt célja egy olyan intranet környezet kialakítása, amely **hallgatói és oktatói** igényeket szolgál ki: órarend megjelenítés, jegyek és hiányzások kezelése, belső információk megosztása. A megoldás **virtualizált** infrastruktúrán működik, így könnyen tesztelhető, menthető és bővíthető.

**Kulcselvek:**
- Központi jogosultságkezelés (AD-csoportok: Diák, Tanár, Admin).
- Hálózati szeparáció és tűzfal-szabályozás.
- Minimális jogosultság elve az alkalmazás és az adatbázis között.
- Egyszerű üzemeltethetőség és menthetőség.

---

## 2. Rendszer-architektúra
- **Proxmox VE** - hypervisor, több VM futtatása (OPNsense, Windows, SQL).  
- **OPNsense** - tűzfal/router, DHCP a termi hálókon, forgalomszabályozás.  
- **Windows Server (AD/DNS/IIS)** - címtár, névfeloldás, webkiszolgáló.  
- **SQL Server** - külön adatbázis-szerver (1433/TCP), csak a webalkalmazás éri el.  
- **ASP.NET webapp** - IIS-en futó alkalmazás belső hostnévvel (pl. `web.intranet.local`).

**Erőforrás-javaslat (kezdeti):**
- OPNsense: 2 vCPU, 2-4 GB RAM
- Windows AD/DNS/IIS: 4 vCPU, 8-16 GB RAM
- SQL Server: 4 vCPU, 8-16 GB RAM, gyors háttértár

---

## 3. Hálózati topológia és DNS
**Proxmox bridge-ek:**
- `vmbr0` - WAN/uplink
- `vmbr1` - SERVER háló (AD/DNS/IIS/SQL - statikus IP-k)
- `vmbr2` - CLASS_A (DHCP tartomány)
- `vmbr3` - CLASS_B (DHCP tartomány)

**OPNsense interfészek:**
- WAN → `vmbr0`
- SERVER → `vmbr1`
- CLASS_A → `vmbr2` (DHCP: `10.10.1.100-10.10.1.200`)
- CLASS_B → `vmbr3` (DHCP: `10.10.2.100-10.10.2.200`)

**DNS:**
- Belső zóna: `intranet.local`
- Rekordok: `web.intranet.local` (IIS), `db.intranet.local` (SQL)

**Fő tűzfalszabályok:**
- CLASS_* → Web (80/443) **ALLOW**
- CLASS_* → SQL (1433) **DENY**
- Web (IIS) → SQL (1433) **ALLOW**
- SERVER ↔ AD/DNS/IIS/SQL **ALLOW**

---

## 4. Telepítés lépésről lépésre

### 4.1 Proxmox VE
1. Proxmox telepítése, majd bridge-ek létrehozása (`vmbr0..3`).  
2. VM-ek létrehozása és a megfelelő bridge-ekhez csatolása.  
3. Snapshot készítése nagyobb változások előtt.

### 4.2 OPNsense
1. Interfészek hozzárendelése: WAN/SERVER/CLASS_A/CLASS_B.  
2. DHCP engedélyezése csak a termi hálókon.  
3. Tűzfalszabályok felvétele (lásd fent).  
4. DNS forwarderek és NTP beállítása a domainhez igazítva.

### 4.3 Windows Server - AD/DNS/IIS
1. AD DS + DNS szerepkör telepítése, tartomány: `intranet.local`.  
2. Csoportok: `Diak`, `Tanar`, opcionálisan `Admin`.  
3. IIS és .NET Hosting Bundle telepítése.  
4. DNS rekordok felvétele: `web.intranet.local`, `db.intranet.local`.  
5. HTTPS bekötése belső CA vagy saját tanúsítvánnyal.

### 4.4 SQL Server
1. SQL Server 2022 telepítése külön VM-re.  
2. Mixed Mode autentikáció (erős jelszavak).  
3. 1433/TCP elérhetőség: **csak** az IIS szerver IP-jének engedélyezve az OPNsense-en.  
4. Alapséma és mintaadatok betöltése (lásd [5. fejezet](#5-adatbázis-séma-minta)).  
5. Külön alkalmazás-felhasználó minimális jogosultságokkal.

### 4.5 ASP.NET webalkalmazás
1. `dotnet publish -c Release` a projektben.  
2. IIS-ben új webhely, hostnév: `web.intranet.local`, HTTPS kötés.  
3. Connection string beállítása `db.intranet.local,1433`-ra.  
4. AppPool megfelelő .NET verzióval, naplózás bekapcsolása.

---

## 5. Adatbázis-séma (minta)
```sql
CREATE TABLE Felhasznalok (
    Id INT IDENTITY PRIMARY KEY,
    Nev NVARCHAR(200) NOT NULL,
    Email NVARCHAR(320) NOT NULL UNIQUE,
    Szerep NVARCHAR(50) NOT NULL CHECK (Szerep IN ('Diak','Tanar','Admin')),
    JelszoHash VARBINARY(512) NOT NULL,
    LetrehozvaAt DATETIME2 NOT NULL DEFAULT SYSDATETIME()
);

CREATE TABLE Orarend (
    Id INT IDENTITY PRIMARY KEY,
    KurzusKod NVARCHAR(50) NOT NULL,
    Tantargy NVARCHAR(200) NOT NULL,
    Idopont DATETIME2 NOT NULL,
    Helyszin NVARCHAR(200) NOT NULL
);

CREATE TABLE Jegyek (
    Id INT IDENTITY PRIMARY KEY,
    DiakId INT NOT NULL FOREIGN KEY REFERENCES Felhasznalok(Id),
    Tantargy NVARCHAR(200) NOT NULL,
    Erdemjegy TINYINT NOT NULL CHECK (Erdemjegy BETWEEN 1 AND 5),
    BejegyzesDatuma DATETIME2 NOT NULL DEFAULT SYSDATETIME()
);

CREATE TABLE Hianyzasok (
    Id INT IDENTITY PRIMARY KEY,
    DiakId INT NOT NULL FOREIGN KEY REFERENCES Felhasznalok(Id),
    Datum DATE NOT NULL,
    Ora NVARCHAR(100) NOT NULL,
    Igazolt BIT NOT NULL DEFAULT 0
);
```

---

## 6. Jogosultságkezelés
- **Active Directory csoportok:** `Diak`, `Tanar`, `Admin`.  
- **Alkalmazás-szint:** role-based authorization (ASP.NET).  
- **Hozzáférési elvek:**  
  - Diák: saját jegyek/hiányzások, órarend.  
  - Tanár: saját órarend, diákok hiányzásainak kezelése.  
  - Admin: felhasználókezelés, naplózás.

---

## 7. Biztonsági megfontolások
- Hálózati szeparáció (bridge-ek) és tűzfal-szabályok.  
- HTTPS mindenhol, belső CA tanúsítványokkal.  
- Minimális jogosultság elve (IIS → SQL csak szükséges jogok).  
- Titkok kezelése: ne commitolj jelszavakat, használj környezeti változókat vagy titokkezelőt.  
- Rendszeres frissítések (Windows Update, OPNsense pluginok).

---

## 8. Üzemeltetés: mentés, monitoring
**Backup:**
- SQL: teljes + differenciális mentések, rendszeres visszaállítási próba.  
- AD: System State backup.  
- IIS: konfiguráció és tartalom mentése.  
- Proxmox: VM snapshot/backup, offsite tárolással.

**Monitoring és naplózás:**
- Windows Event Log, IIS naplók, SQL Error/Agent logok.  
- OPNsense: Packet Capture, Firewall Hits.  
- Központosítás (opcionális): Graylog / Elastic Stack.

---

## 9. Tesztelési ellenőrzőlista
- [ ] DNS feloldás: `web.intranet.local`, `db.intranet.local` a termi hálózatokból.  
- [ ] Elérés: Termek → Web/IIS (80/443) engedélyezve.  
- [ ] Közvetlen Termek → SQL tiltott.  
- [ ] Bejelentkezés AD-val; szerepek szerint eltérő menük/nézetek.  
- [ ] Adatbázis integritás (FK/CHK), tranzakciók.  
- [ ] Terhelés: párhuzamos hallgatói lekérdezések.  
- [ ] Mentés-visszaállítás próba (SQL, IIS, AD).

---

## 10. Hibakeresés
- **DNS:** `nslookup web.intranet.local`; ellenőrizd a DNS rekordokat és forwardereket.  
- **AD login:** időszinkron (NTP), SPN-ek, tűzfal-portok (Kerberos/LDAP).  
- **IIS 500/502:** alkalmazásnapló, `web.config`, connection string, AppPool jogosultságok.  
- **SQL kapcsolat:** 1433/TCP elérés IIS-ről, OPNsense szabályok, DB-user jogok.  
- **Hálózat:** OPNsense Rules „hit count”, Packet Capture.

---

## 11. Bővíthetőség és jövőbeli fejlesztések
- **Nagy rendelkezésre állás:** SQL Always On, kétcsomópontos AD.  
- **Reverse proxy/WAF:** OPNsense plugin vagy külön Nginx.  
- **CI/CD:** GitHub Actions a buildre és publikálásra.  
- **Központi naplógyűjtés** és riasztások.  
- **Chat/fájlszerver modulok** hozzáadása a webapphoz.

---


**Verzió:** 1.0 • Szerző: Paleksz
