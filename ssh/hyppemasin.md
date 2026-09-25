# SSH hüppemasin (jump host)

Siin on praeguse SSH-ligipääsu hinnang ja etapiviisiline lahendus ühele tiimile.

Näidetes on hüppemasin `jumpserver` (`jumpserver.firma.ee`, sisevõrgu IP `10.10.0.5`) ja serverid `server1`, `server2` jne asuvad võrgus `10.10.0.0/24`, nimedega kujul `*.sise` (nt `server1.sise`).

## Sisukord

1. [Praegune olukord](#praegune-olukord)
2. [Hinnang](#hinnang)
3. [Milline lahendus millal](#milline-lahendus-millal)
4. [1. etapp: vähe servereid, üks tiim](#1-etapp-vähe-servereid-üks-tiim)
5. [2. etapp: kümneid servereid, mitu inimest](#2-etapp-kümneid-servereid-mitu-inimest)
6. [Olemasolevate võtmete kasutamine](#olemasolevate-võtmete-kasutamine)
7. [Suurem mastaap: sadu servereid, mitu tiimi](#suurem-mastaap-sadu-servereid-mitu-tiimi)
8. [Märkus: WarnWeakCrypto](#märkus-warnweakcrypto)

---

## Praegune olukord

```
   sinu HP (kasutaja@hp)
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
      │  SSH protokoll ja sisselogimine: ikka sinu HP ↔ server1
      │  autentimine: sinu -i ~/.ssh/voti
      ▼
    shell server1 peal
```

Käsk `ssh -J jumpserver1,jumpserver2 server1` töötab nii:

- Kõik kolm SSH-seanssi (HP↔jumpserver1, HP↔jumpserver2, HP↔server1) algavad **sinu HP-st**. Hüppemasinad ainult edastavad TCP-ühendust (`direct-tcpip`, sama mis `ssh -W`).
- Privaatvõti ei lahku kunagi HP-lt. `jumpserver1` ja `jumpserver2` ei näe `server1` seansi sisu, sest see on HP ja server1 vahel otsast lõpuni krüpteeritud.

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

### 1.5 Kui jumpserver on maas

- Hädaolukorra jaoks on olemas **konsooliligipääs** serveritele (hüperviisor või IPMI). Selle paroolid hoitakse paroolihalduris.
- Varu: jumpserveri konfiguratsioon on kirjas (või skriptina olemas), et uue hüppemasina saaks tunniga püsti.

### 1.6 Töökorraldus

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
