# Tiimipõhine SSH hüppemasin (jump server)

Pöördumine IT-osakonnale. Praegu käib meie tiimi töö serveritega peamiselt nii, et kõigepealt logitakse RDP-ga Windowsi terminaliserverisse ja sealt edasi PuTTY-ga serveritesse. **Soovime sellest vaheastmest loobuda.** Selle asemel tahame tiimile eraldi SSH hüppemasinat (jump server), mille kaudu saab tiimi serveritesse ühenduda otse oma tööjaamast.

Allpool on kirjas praegune olukord, selle probleemid, mida soovime ja miks. Dokumendi teises pooles on tehniline lahendusettepanek, millest saab lähtuda.

## Sisukord

1. [Probleemipüstitus](#probleemipüstitus)
2. [Praegune olukord tehniliselt](#praegune-olukord-tehniliselt)
3. [Hinnang](#hinnang)
4. [Milline lahendus millal](#milline-lahendus-millal)
5. [1. etapp: vähe servereid, üks tiim](#1-etapp-vähe-servereid-üks-tiim)
6. [2. etapp: kümneid servereid, mitu inimest](#2-etapp-kümneid-servereid-mitu-inimest)
7. [Olemasolevate võtmete kasutamine](#olemasolevate-võtmete-kasutamine)
8. [Suurem mastaap: sadu servereid, mitu tiimi](#suurem-mastaap-sadu-servereid-mitu-tiimi)
9. [Märkus: WarnWeakCrypto](#märkus-warnweakcrypto)

---

## Probleemipüstitus

### Taust

- Meil on **üks tiim**, kus on mitu inimest.
- Tiim haldab **kümneid Linuxi servereid**.
- Enamik tööd tehakse praegu **RDP kaudu Windowsi terminaliserveris, kust edasi PuTTY-ga** serveritesse.

### Kuidas see praegu käib

**Peamine viis: RDP ja PuTTY**

```
   tiimiliikme tööjaam
      │  RDP
      ▼
   Windowsi terminaliserver     ← graafiline seanss, PuTTY, võtmed/paroolid
      │  PuTTY (SSH)
      ▼
   server1, server2, ...        ← töö käib siin
```

**Teine võimalus: otse SSH kahe hüppemasina kaudu.** Tehniliselt on see juba praegu võimalik, aga igapäevaselt seda ei kasutata:

```
   tööjaam (kasutaja@hp)
      │  ssh -J jumpserver1,jumpserver2 server1
      │  võti: ~/.ssh/voti
      ▼
   [1] jumpserver1     ← esimene hüpe (config: ProxyJump puudub)
      │  (sinu võti → kasutaja@jumpserver1)
      ▼
   [2] jumpserver2     ← teine hüpe
      │  (jälle sinu võti → kasutaja@jumpserver2)
      ▼
   [3] server1:22
      │  SSH protokoll ja sisselogimine: ikka tööjaam ↔ server1
      │  autentimine: sinu -i ~/.ssh/voti
      ▼
    shell server1 peal
```

Mõlemal juhul peab iga inimese avalik võti olema failis `authorized_keys` kõigis masinates, kuhu ta ligi pääseb.

### Probleemid

**RDP ja terminaliserveri vaheaste (peamine probleem):**

1. **Töö on aeglane ja ebamugav.** Iga toiming käib läbi kaugtöölaua: klaviatuuri viivitus, lõikelaua ja kodeeringu probleemid, akende haldus kahes kohas.
2. **Tavalised tööriistad ei tööta otse.** `scp`/`rsync`, `git`, Ansible, VS Code Remote-SSH, skriptid ja automaatika eeldavad SSH ühendust otse tööjaamast. Praegu tuleb failid ja käsud tõsta läbi terminaliserveri.
3. **Terminaliserver on suur turvarisk.** Seal on korraga mitme inimese seansid ja tõenäoliselt ka SSH võtmed või salvestatud PuTTY seansid. Kui masin kompromiteeritakse, saab ründaja ligipääsu kõigile, kes seda kasutavad. Windowsi mälust saab kasutajate andmeid kätte (nt `mimikatz`, Pageant'i mälu).
4. **Privaatvõti ei ole kasutaja kontrolli all.** Kui võti asub jagatud serveris, ei ole see enam isiklik.
5. **Veel üks süsteem, mida hallata ja litsentseerida.** Windows Serveri uuendused, RDS litsentsid (CAL), PuTTY versioonid, kasutajaprofiilid.
6. **Veel üks rikkekoht.** Kui terminaliserver on maas, ei saa keegi serveritega töötada.

**Ligipääsude haldus ja piiramine:**

7. **Ligipääsu haldamine on käsitsi ja paljudes kohtades.** Uue inimese lisamisel tuleb tema võti panna kümnetesse `authorized_keys` failidesse. Inimese lahkumisel tuleb see kõigist uuesti eemaldada. Tavaliselt jääb mõni vahele, ja see vana võti jääbki kehtima.
8. **Puudub ülevaade, kellel kuhu ligipääs on.** Selle teadasaamiseks tuleb käia läbi kõik serverid.
9. **Võtmed ei aegu.** Kord lisatud võti kehtib igavesti, kui keegi seda ise ei eemalda.
10. **Ligipääs ei ole piiratud tiimi serveritega.** Hüppemasinad ei ole tiimipõhised. Seega sõltub ainult serverite endi seadistusest, kuhu nende kaudu pääseb.
11. **Sisselogimisi on raske auditeerida.** Logid on laiali kõigis masinates. Jagatud kasutajakonto (`kasutaja`) või terminaliserveri kaudu tulles ei ole logist kohe näha, kes tegelikult sisse logis.
12. **Kaks hüpet lisavad keerukust.** Kui `jumpserver1` ja `jumpserver2` ei asu erinevates võrgutsoonides, ei lisa teine hüpe turvalisust, küll aga haldustööd ja ühe rikkekoha juurde.
13. **Serverite seadistus erineb.** Osal serveritest on vana OpenSSH, mis ei toeta kaasaegset krüptot. Klienti tuleb hoiatuste vaigistamiseks eraldi seadistada (`WarnWeakCrypto no-pq-kex`, vt [märkust](#märkus-warnweakcrypto)).

### Mida soovime

1. **Loobuda RDP ja terminaliserveri vaheastmest** SSH töö jaoks.
2. **Tiimipõhist hüppemasinat:** üks `jumpserver`, mis on mõeldud ainult meie tiimile ja mille kaudu pääseb ainult meie tiimi serveritele.
3. **SSH ühendust otse oma tööjaamast.** Windowsis on OpenSSH klient sisse ehitatud, samuti on olemas Windows Terminal. Sobivad ka PuTTY, WinSCP ja VS Code, kõik oskavad hüppemasinat kasutada.

```
 tiimiliikme tööjaam ──► jumpserver (ainult edastab) ──► tiimi serverid
   isiklik võti                                        (SSH ainult jumpserverist)
   (Windows Terminal / OpenSSH, PuTTY, VS Code)
```

### Mida see lahendab

| Probleem praegu | Lahendus tiimipõhise hüppemasinaga |
|---|---|
| Töö käib läbi RDP kaugtöölaua | SSH otse tööjaamast, ilma graafilise vaheastmeta |
| Tööriistad (`scp`, `git`, Ansible, VS Code) ei tööta otse | Kõik tööriistad töötavad tööjaamast läbi `ProxyJump`-i |
| Terminaliserveris on mitme inimese seansid ja võtmed | Terminaliserverit pole SSH töö jaoks vaja. Iga võti asub ainult omaniku tööjaamas (soovitatavalt parooliga või riistvaravõtmel) |
| Hallata tuleb Windows Serverit ja RDS litsentse | `jumpserver` on väike Linuxi masin, mis ainult edastab ühendusi |
| Võtmed on kümnetes `authorized_keys` failides | SSH sertifikaadid: serverites on üks CA võti ja ligipääs antakse ühes kohas |
| Lahkunud töötaja võti jääb kehtima | Sertifikaat aegub ise, vajadusel saab selle tsentraalselt tühistada |
| Pole ülevaadet, kellel kuhu ligipääs on | Ligipääs on kirjas sertifikaadis (principal/roll) ja väljastatud sertifikaatide nimekirjas |
| Ligipääs ei ole tiimiga piiratud | `jumpserver` pääseb tulemüüri järgi ainult tiimi võrku ja serverid võtavad SSH ühendusi vastu ainult `jumpserver`-ist |
| Logist ei näe, kes sisse logis | Sertifikaadis on inimese nimi ja see jõuab iga serveri logisse. Logid saadetakse keskselt kogumiseks |
| Kaks hüpet | Üks hüpe, välja arvatud juhul, kui võrgutsoonid seda tegelikult nõuavad |
| Serverite seadistus erineb | Kõik seadistused tehakse ühest Ansible rollist, nii on OpenSSH versioon ja `sshd_config` kõikjal ühesugused |

### Ootused lahendusele

- [ ] Tiimi liikmed saavad tiimi serveritesse SSH-ga otse oma tööjaamast, ilma RDP-ta.
- [ ] Töötab Windowsi sisseehitatud OpenSSH kliendiga (Windows Terminal) ja PuTTY-ga. Soovitavalt töötavad ka WinSCP ja VS Code Remote-SSH.
- [ ] Tiimil on üks oma hüppemasin (`jumpserver`), mis ainult edastab ühendusi, ilma shellita.
- [ ] Tööjaamade võrgust on lubatud ühendus `jumpserver`-i porti 22 (vajadusel ainult VPN-i kaudu).
- [ ] `jumpserver` saab ühenduda ainult tiimi serverite võrku (väljuv tulemüür).
- [ ] Tiimi serverid lubavad SSH ühendusi ainult `jumpserver`-ist (sisenev tulemüür).
- [ ] Sisselogimine käib isikliku võtmega, paroolid on keelatud. Jagatud võtmeid ei kasutata.
- [ ] Ligipääs antakse SSH sertifikaatidega, millel on piiratud kehtivus. Inimese lisamiseks ega eemaldamiseks ei pea serverites midagi muutma.
- [ ] Logist on näha, kes (inimese nimi) millal kuhu sisse logis, ja logid on koondatud ühte kohta.
- [ ] Hostivõtmed on kontrollitavad (hostisertifikaadid või tiimile jagatud `known_hosts` fail).
- [ ] Seadistus on koodina (Ansible vms): uue serveri saab sama seadistusega üles panna ja `jumpserver`-i saab rikke korral kiiresti uuesti ehitada.
- [ ] Hädaolukorraks on olemas konsooliligipääs serveritele, kui `jumpserver` on maas.
- [ ] Kui uus lahendus töötab, eemaldatakse terminaliserverist tiimi SSH võtmed ja PuTTY seansid, ning SSH ligipääs serveritesse terminaliserverist suletakse.

### Mida tiim omalt poolt annab

- tiimi serverite nimekirja (nimed ja IP-d);
- tiimi liikmete nimekirja ja nende avalikud võtmed (olemasolevaid võtmeid saab sertifikaatide jaoks edasi kasutada, vt [olemasolevate võtmete kasutamine](#olemasolevate-võtmete-kasutamine)). Kui võti on praegu ainult terminaliserveris, teeb inimene oma tööjaamas uue võtme;
- vajadusel rollid, kui kõik ei pea pääsema kõigile serveritele (nt `web`, `db`);
- testimise üleminekul, kus vana ja uus lahendus töötavad paralleelselt.

### Küsimused IT-osakonnale

1. Kas tööjaamadest on võimalik lubada SSH ühendus (port 22) hüppemasinasse? Kas see peab käima VPN-i kaudu?
2. Kas tööjaamad on hallatud (nt Intune) ja kas neil on lubatud Windowsi OpenSSH klient ja `ssh-agent` teenus?
3. Kas terminaliserverit kasutatakse ka millekski muuks peale SSH? Kas seda saab tiimi jaoks pärast üleminekut sulgeda?
4. Kas tiimi serverid on eraldi võrgus (VLAN või alamvõrk), või tuleb see teha?
5. Miks on praegu kaks hüpet? Kas `jumpserver1` ja `jumpserver2` asuvad erinevates võrgutsoonides?
6. Kas ettevõttes on juba olemas SSH CA või SSO (nt Entra ID, Keycloak), millega sertifikaatide väljastamise saaks siduda?
7. Kas serverite seadistamiseks kasutatakse juba Ansible'it või mõnda muud tööriista?
8. Kas eelistate ise ehitatud lahendust (OpenSSH + CA) või valmis platvormi (Teleport, Tailscale SSH, HashiCorp Boundary)?

Tehniline lahendusettepanek on allpool. Meie olukorda (üks tiim, kümned serverid, mitu inimest) sobib [2. etapp](#2-etapp-kümneid-servereid-mitu-inimest). [1. etapi](#1-etapp-vähe-servereid-üks-tiim) saab teha esimese sammuna ja see on 2. etapi aluseks. Windowsi tööjaama seadistus on kirjas punktis [1.5](#15-windowsi-tööjaam).

---

## Praegune olukord tehniliselt

Näidetes on hüppemasin `jumpserver` (`jumpserver.firma.ee`, sisevõrgu IP `10.10.0.5`) ja serverid `server1`, `server2` jne asuvad võrgus `10.10.0.0/24`, nimedega kujul `*.sise` (nt `server1.sise`).

Otse-SSH ahel on joonisel [probleemipüstituses](#kuidas-see-praegu-käib).

Käsk `ssh -J jumpserver1,jumpserver2 server1` töötab nii:

- Kõik kolm SSH-seanssi (tööjaam↔jumpserver1, tööjaam↔jumpserver2, tööjaam↔server1) algavad **tööjaamast**. Hüppemasinad ainult edastavad TCP-ühendust (`direct-tcpip`, sama mis `ssh -W`).
- Privaatvõti ei lahku kunagi tööjaamast. `jumpserver1` ja `jumpserver2` ei näe `server1` seansi sisu, sest see on tööjaama ja server1 vahel otsast lõpuni krüpteeritud.

Sama ahel `~/.ssh/config` failis:

```sshconfig
Host jumpserver1
    HostName jumpserver1.firma.ee
    User kasutaja
    IdentityFile ~/.ssh/voti
    IdentitiesOnly yes

Host jumpserver2
    HostName jumpserver2.firma.ee
    User kasutaja
    IdentityFile ~/.ssh/voti
    IdentitiesOnly yes
    ProxyJump jumpserver1

Host server1
    HostName server1.sise
    User kasutaja
    IdentityFile ~/.ssh/voti
    IdentitiesOnly yes
    ProxyJump jumpserver2
    WarnWeakCrypto no-pq-kex
```

---

## Hinnang

### Mis on hästi

- **`-J` / `ProxyJump` on õige tehnika.** Seanss on otsast lõpuni krüpteeritud, privaatvõti ei lahku sinu arvutist ja agendi edastamist (`ForwardAgent`) ei kasutata.
- **Hüppemasinad annavad võrgu segmenteerimise.** Sisevõrgu masinatele ei saa otse ligi.

### Mis on nõrk, kui servereid on palju

| Probleem | Miks see halb on |
|---|---|
| **Üks võti kõikjal** | Kui võti lekib, on ründajal ligipääs kõigile serveritele. |
| **Kõigis serverites `authorized_keys`** | Kui töötaja lahkub, tuleb tema võti eemaldada kõigist serveritest. Tavaliselt jääb mõni vahele. |
| **Võtmed ei aegu** | Viis aastat tagasi lisatud võti kehtib endiselt. |
| **MFA puudub** | Ligipääsuks piisab failist kettal. |
| **Hostivõtmeid usaldatakse esimesel ühendusel (TOFU)** | Kõik vajutavad lihtsalt „yes“, nii et MITM-rünnak jääb märkamata. |
| **Keskset auditit ei ole** | Logid on laiali kõigis masinates. |
| **Hüppemasinates on shell** | Hüppemasin on kõige väärtuslikum sihtmärk. |
| **Vanad `sshd` versioonid** | Uuendusi ei hallata tsentraalselt. |

### Kas on vaja kahte hüpet?

Kahe hüppemasina ahel (`jumpserver1` → `jumpserver2`) on mõistlik ainult siis, kui need asuvad **erinevates võrgutsoonides**, näiteks jumpserver1 on DMZ-s ja jumpserver2 sisevõrgu servas ning jumpserver1-st otse server1-ni ei pääse. Kui mõlemad on sisuliselt samas tsoonis, lisab teine hüpe haldustööd ja viivitust, aga turvalisust mitte. Siis piisaks ühest.

---

## Milline lahendus millal

| Serverite arv | Soovitus |
|---|---|
| Vähe, üks tiim | Üks hüppemasin koos võtmetega, tulemüür lubab SSH ainult hüppemasinast ([1. etapp](#1-etapp-vähe-servereid-üks-tiim)) |
| Kümneid, mitu inimest | Üks hüppemasin koos SSH sertifikaatidega ([2. etapp](#2-etapp-kümneid-servereid-mitu-inimest)) |
| Sadu, mitu tiimi | Sertifikaadid principal'idega, vajadusel tiimipõhised võrgutsoonid ([suurem mastaap](#suurem-mastaap-sadu-servereid-mitu-tiimi)) |

2. etapp ei vaheta midagi välja, vaid lisab 1. etapile Ansible'i ja sertifikaadid. Nii ei pea hiljem midagi ümber ehitama.

---

## 1. etapp: vähe servereid, üks tiim

```
 tiimiliikme HP ──► jumpserver (ainult edastab) ──► serverid (SSH ainult jumpserverist)
   isiklik võti      kasutaja "jump"            kasutaja "admin"
```

### 1.1 Igal inimesel on oma võti

```bash
ssh-keygen -t ed25519 -f ~/.ssh/voti -C "mari.maasikas"
```

Võti peab olema parooliga ja iga inimene kasutab ainult oma võtit. Jagatud võtmeid ei tohi olla. Kommentaar (`-C`) on inimese nimi, siis on `authorized_keys` failist kohe näha, kellele võti kuulub.

### 1.2 Hüppemasin `jumpserver`

`/etc/ssh/sshd_config`:

```sshconfig
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
LogLevel VERBOSE                 # logisse jääb võtme sõrmejälg, nii on näha, kes sisse logis
AllowUsers jump

Match User jump
    AllowTcpForwarding yes
    PermitOpen *:22
    PermitTTY no
    X11Forwarding no
    AllowAgentForwarding no
    ForceCommand /usr/sbin/nologin
```

Faili `/home/jump/.ssh/authorized_keys` tuleb iga tiimiliikme kohta üks rida:

```
restrict,port-forwarding ssh-ed25519 AAAA... mari.maasikas
restrict,port-forwarding ssh-ed25519 AAAA... jaan.tamm
```

Tulemüür lubab jumpserverist väljuvat SSH liiklust ainult serverite võrku. `PermitOpen` ei tunne alamvõrke (CIDR), seetõttu tuleb see piirang teha tulemüüris:

```
# /etc/nftables.conf (jumpserver) – output chain
ct state established,related accept
ip daddr 10.10.0.0/24 tcp dport 22 accept
udp dport 53 accept              # DNS
udp dport 123 accept             # NTP
drop
```

Lisaks lülita sisse automaatsed turvauuendused (`unattended-upgrades` või `dnf-automatic`). Muud tarkvara jumpserverisse ei paigaldata.

### 1.3 Serverid

`/etc/ssh/sshd_config`:

```sshconfig
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
LogLevel VERBOSE
AllowUsers admin
```

Faili `/home/admin/.ssh/authorized_keys` pannakse samad tiimi võtmed (ilma `restrict` eesliiteta). Kasutaja `admin` saab vajadusel `sudo` õiguse.

Tulemüür lubab SSH ühendusi **ainult jumpserverist**. See on kõige tähtsam reegel, sest muidu saab hüppemasinast lihtsalt mööda minna:

```
# input chain
ip saddr 10.10.0.5 tcp dport 22 accept
tcp dport 22 drop
```

### 1.4 Kliendi config (sama fail kõigil tiimiliikmetel)

`~/.ssh/config`:

```sshconfig
Host jumpserver
    HostName jumpserver.firma.ee
    User jump
    IdentityFile ~/.ssh/voti
    IdentitiesOnly yes

Host *.sise
    User admin
    ProxyJump jumpserver
    IdentityFile ~/.ssh/voti
    IdentitiesOnly yes
```

Kasutamine: `ssh server1.sise` või `ssh server2.sise`

Hostivõtmed kogu üks kord kokku ja jaga tiimile ühise failina, siis ei pea keegi esimesel ühendusel pimesi „yes“ vastama:

```bash
ssh-keyscan jumpserver.firma.ee > known_hosts.tiim
# serverite võtmed korja serverite endi pealt: /etc/ssh/ssh_host_ed25519_key.pub
```

Soovi korral lisa ühenduste taaskasutus, et iga ühendus ei teeks uuesti kõiki käepigistusi:

```sshconfig
Host *
    ControlMaster auto
    ControlPath ~/.ssh/cm-%C
    ControlPersist 10m
```

### 1.5 Windowsi tööjaam

RDP ja terminaliserverit pole vaja. Kõik allolev toimub tiimiliikme enda tööjaamas.

**Windowsi sisseehitatud OpenSSH (Windows Terminal / PowerShell)**

OpenSSH klient on Windows 10 (1809+) ja Windows 11 osa. Kontroll:
```powershell
ssh -V
```

Võti tehakse oma tööjaamas, parooliga:
```powershell
ssh-keygen -t ed25519 -f $env:USERPROFILE\.ssh\voti -C "mari.maasikas"
```

`ssh-agent` teenus hoiab võtit mälus, nii et parooli ei pea iga kord sisestama:
```powershell
# administraatorina, üks kord
Get-Service ssh-agent | Set-Service -StartupType Automatic
Start-Service ssh-agent

# tavakasutajana
ssh-add $env:USERPROFILE\.ssh\voti
```

Config-fail on `%USERPROFILE%\.ssh\config`, sama sisuga nagu [punktis 1.4](#14-kliendi-config-sama-fail-kõigil-tiimiliikmetel). Windowsi OpenSSH ei toeta `ControlMaster` valikut, seega jäta see plokk Windowsis välja.

Kasutamine Windows Terminalis või PowerShellis:
```powershell
ssh server1.sise
scp .\fail.txt server1.sise:/tmp/
```

**PuTTY (0.77 või uuem)**

- *Session:* Host Name `server1.sise`
- *Connection → Data:* Auto-login username `admin`
- *Connection → Proxy:* Proxy type **SSH to proxy and use port forwarding**, Proxy hostname `jumpserver.firma.ee`, Port `22`, Username `jump`
- *Connection → SSH → Auth → Credentials:* privaatvõti `.ppk` kujul (OpenSSH võtme saab teisendada PuTTYgen'iga). 2. etapi sertifikaadi saab lisada väljale *Certificate to use with the private key* (PuTTY 0.78+).
- Salvesta seanss. Pageant hoiab võtit mälus.

**WinSCP:** *Session → Advanced → Connection → Tunnel*. Märgi „Connect through SSH tunnel“ ja sisesta host `jumpserver.firma.ee`, kasutaja `jump` ja sama võti.

**VS Code Remote-SSH** kasutab faili `%USERPROFILE%\.ssh\config` otse, seega `ProxyJump` töötab ilma lisaseadistuseta.

**Oluline:** privaatvõti asub **ainult omaniku tööjaamas**. Seda ei kopeerita terminaliserverisse, võrgukettale ega kellegi teise arvutisse.

### 1.6 Kui jumpserver on maas

- Hädaolukorra jaoks on olemas **konsooliligipääs** serveritele (hüperviisor või IPMI). Selle paroolid hoitakse paroolihalduris.
- Varu: jumpserveri konfiguratsioon on kirjas (või skriptina olemas), et uue hüppemasina saaks tunniga püsti.

### 1.7 Töökorraldus

| Sündmus | Tegevus |
|---|---|
| Uus inimene | Tema `.pub` rida lisatakse jumpserveri ja serverite `authorized_keys` failidesse |
| Inimene lahkub | Tema rida eemaldatakse **kõigist** failidest, `grep "mari.maasikas"` abil |
| Kord kvartalis | `authorized_keys` failid vaadatakse üle: kas iga rea omanik on veel teada? |

### Mida mitte teha

- **`ForwardAgent yes`** või `ssh jumpserver1`, siis seal `ssh jumpserver2`. Nii satub agendi sokkel hüppemasinasse ja sealne root saab sinu võtit kasutada. Õige viis on `-J` / `ProxyJump`.
- **Jagatud võtmed.** Logist ei saa aru, kes tegelikult sisse logis, ja inimese lahkumisel tuleb võti kõigil vahetada.

---

## 2. etapp: kümneid servereid, mitu inimest

**Millal üle minna:** kui `authorized_keys` failide käsitsi uuendamine hakkab ununema, kui inimesed tulevad ja lähevad tihti või kui kõigil ei tohi olla ligipääs kõigile serveritele.

Hüppemasin, tulemüürid ja kliendi config jäävad samaks. Muutub kolm asja.

### 2.1 Ansible

Kogu 1. etapi seadistus (sshd_config, tulemüür, uuendused) pannakse ühte Ansible rolli. Nii on kõik serverid ühesugused ja uus server saab seadistuse ühe käsuga. Ilma selleta ei ole mõtet 2. etapiga edasi minna.

### 2.2 SSH sertifikaadid võtmete asemel

**CA (üks kord, turvalises masinas):**

```bash
ssh-keygen -t ed25519 -f user_ca -C "firma user CA"
```

Faili `user_ca` hoia väga hästi (parooliga, võrguühenduseta masinas või YubiKeyl). Fail `user_ca.pub` läheb kõigisse serveritesse.

**Serveritesse ja jumpserverisse** (Ansible'iga):

```sshconfig
TrustedUserCAKeys /etc/ssh/user_ca.pub
AuthorizedPrincipalsFile /etc/ssh/principals/%u
```

- jumpserveris on failis `/etc/ssh/principals/jump` rida `tiim`;
- serverites on failis `/etc/ssh/principals/admin` rida `tiim` või täpsem roll, näiteks `db`, `web` või `admin`.

**Kasutajate olemasolevate võtmete allkirjastamine:**

```bash
ssh-keygen -s user_ca -I mari.maasikas -n tiim,web -V +30d mari.pub
```

- `-I` on nimi, mis ilmub serveri logidesse.
- `-n` on principal'id ehk rollid, kuhu inimene pääseb.
- `-V +30d` tähendab, et sertifikaat kehtib 30 päeva ja aegub siis ise.

Mari paneb saadud faili nimega `~/.ssh/voti-cert.pub`. Tema configis ei muutu midagi.

**Üleminek:** vanad võtmed ja sertifikaadid töötavad paralleelselt. Kui logides on näha `ID mari.maasikas` kõigilt kasutajatelt, tühjenda `authorized_keys` failid (või `AuthorizedKeysFile none`). Enne seda jäta endale kindlasti hädaabi-ligipääs.

| Sündmus | 1. etapp | 2. etapp |
|---|---|---|
| Uus inimene | Muuta tuleb N faili | Allkirjasta üks sertifikaat |
| Inimene lahkub | Eemalda N failist | Sertifikaat aegub ise, vajadusel `RevokedKeys` |
| Ligipääs osadele serveritele | Ei ole võimalik | Principal'id (`web`, `db`) |
| Logis on näha | Võtme sõrmejälg | Inimese nimi |

### 2.3 Hostisertifikaadid

Nendega kaob esimesel ühendusel tulev „yes/no“ küsimus ja `known_hosts` faile ei pea enam haldama:

```bash
ssh-keygen -t ed25519 -f host_ca -C "firma host CA"
ssh-keygen -s host_ca -I server1 -h -n server1.sise /etc/ssh/ssh_host_ed25519_key.pub
```

Serverisse lisatakse `HostCertificate /etc/ssh/ssh_host_ed25519_key-cert.pub`. Kliendi `known_hosts` faili tuleb üks rida:

```
@cert-authority *.sise,jumpserver.firma.ee ssh-ed25519 AAAA...host_ca
```

### 2.4 Soovi korral hiljem

- **Teine hüppemasin**, mis on jumpserveriga identne ja tehakse samast Ansible rollist. Siis ei tähenda jumpserveri rike seisakut.
- **Tsentraalne logimine:** jumpserveri ja serverite `sshd` logid saadetakse logiserverisse, kuhu tiimil endal kirjutusõigust pole.
- **Lühem kehtivus ja automaatne väljastamine** läbi SSO+MFA (Smallstep step-ca, HashiCorp Vault SSH), et CA faili ei peaks käsitsi kasutama.
- **FIDO2-võtmed** (`ed25519-sk`), et privaatvõtit ei saaks arvutist varastada.

---

## Olemasolevate võtmete kasutamine

Sertifikaat on alati tehtud olemasoleva avaliku võtme peale. Kasutaja privaatvõti jääb samaks ja tema arvutis ei muutu midagi. Vaja on ainult tema avalikku võtit (`.pub`), mis on `authorized_keys` failides niikuinii olemas.

### Enne allkirjastamist vaata võtmed üle

```bash
ssh-keygen -l -f mari.pub
# 256 SHA256:abc... mari@hp (ED25519)
```

| Mida leiad | Mida teha |
|---|---|
| `ED25519`, `ECDSA`, `RSA` 3072+ | Sobib, võib allkirjastada |
| `RSA` 2048 | Töötab, aga palu uus võti teha |
| `RSA` < 2048, `DSA` | **Ära allkirjasta.** DSA tugi on OpenSSH 10-st eemaldatud |
| **Sama võti mitmel inimesel** (jagatud võti) | **Ära allkirjasta.** Igaüks peab tegema oma võtme |
| Võti, mille omanikku keegi ei tea | **Ära allkirjasta.** Tõenäoliselt kuulub see endisele töötajale või vanale skriptile |
| Automaatika või skriptide võtmed | Anna eraldi principal (nt `deploy`) ja piira seda |

Jagatud võtmeid leiad niimoodi: kogu kõigist serveritest `authorized_keys` read kokku ja otsi sõrmejälgede kordusi:

```bash
cat kogutud/*.authorized_keys | ssh-keygen -l -f - | sort | uniq -c | sort -rn
```

### Lekkinud võti

Sertifikaat on seotud just selle võtmega. Kui kellegi privaatvõti lekib, siis uue sertifikaadi väljastamine ei aita. Tuleb teha uus võti ja vana tühistada:

```sshconfig
RevokedKeys /etc/ssh/revoked_keys
```

```bash
ssh-keygen -k -f revoked_keys mari.pub   # loob uue nimekirja; olemasolevale lisamiseks lisa -u
```

Seda faili jagatakse samuti Ansible'iga. Lühike kehtivus (30 päeva või vähem) tähendab, et seda läheb harva vaja.

---

## Suurem mastaap: sadu servereid, mitu tiimi

Tiimipõhised hüppemasinad on mõistlikud ainult **koos** sertifikaatide ja väljuva liikluse tulemüüriga. Ilma nendeta tekitavad need rohkem haldustööd, aga turvalisus paraneb vähe.

Piirang peab kehtima kolmel tasandil. Siis ei piisa ründajale ühest veast:

```
              SSO + MFA
                  │
                  ▼
           [ SSH CA ]  → sertifikaat: principals = team-a, kehtib 12h
                  │
   HP ──► bastion-team-a ──► team-a serverid
          │                   │
          │ 1) sshd: lubab     │ 3) sshd: lubab ainult
          │    principal       │    principal team-a
          │    team-a          │
          │ 2) tulemüür:       │
          │    väljuv ainult   │
          │    team-a võrku    │
```

- Tiimidevaheline ligipääs (nt DBA või SRE) lahendatakse sertifikaadi principal'iga, mitte uute võtmete või hüppemasinatega.
- Kõik hüppemasinad ehitatakse ühest Ansible rollist. Tiimipõhised on ainult muutujad: principal ja lubatud võrk.

**Valmis platvormid**, mis lahendavad sertifikaadid, SSO/MFA ja auditi korraga: Teleport, HashiCorp Boundary, Tailscale SSH. Nende puhul on vähem ise ehitamist, aga rohkem sõltuvust ühest tootest.

---

## Märkus: WarnWeakCrypto

`WarnWeakCrypto` on OpenSSH 10.1-s lisandunud kliendi valik. Uuem klient hoiatab, kui ühendus ei kasuta post-kvant-võtmevahetust (`mlkem768x25519-sha256` või `sntrup761x25519-sha512`). Rida `WarnWeakCrypto no-pq-kex` lülitab selle hoiatuse välja.

- Hoiatus tähendab, et sihtmasina `sshd` on vana (alla OpenSSH 9.0). Õige lahendus on seda uuendada, soovitatavalt 9.9 või uuemale, ja siis saab selle rea eemaldada.
- Iga hüpe on eraldi SSH-seanss. Valik kehtib ainult selles `Host`-plokis, kuhu see on kirjutatud.
- Vanem klient (alla 10.1) ei tunne seda valikut ja annab vea. Sellisel juhul pane config-faili algusse `IgnoreUnknown WarnWeakCrypto`.

Kontroll:

```bash
ssh -v server1 exit 2>&1 | grep -i "kex: algorithm"
ssh -G server1 | grep -i warnweak
```
