# Viikko 2 – SNMP ja verkon perustason valvonta

## 1. Johdanto

### Mikä on SNMP?
SNMP (Simple Network Management Protocol) on laajasti käytetty sovellustason tietoliikenneprotokolla, joka on suunniteltu TCP/IP-pohjaisten tietoverkkojen laitteiden ja palvelimien valvontaan sekä hallintaan. SNMP toimii asiakas-palvelin-mallilla (Manager-Agent), jossa keskitetty valvontajärjestelmä (Manager) kysyy tietoja valvottavissa laitteissa pyöriviltä taustaprosesseilta (Agent). Tiedonsiirto perustuu pääasiassa keveään ja nopeaan UDP-protokollaan (portti 161).

### Mihin SNMP:tä käytetään verkonhallinnassa?
Verkonhallinnassa SNMP:tä käytetään pääasiassa kolmeen tarkoitukseen:
1. **Tilan ja metriikan seurantaan (Monitoring):** Kerätään laitteistotietoja, kuten prosessorin (CPU) käyttöastetta, muistinkulutusta, levytilaa ja verkkorajapintojen liikennemääriä.
2. **Konfigurointiin (Management):** Muutetaan laitteiden asetuksia etänä (esim. suljetaan kytkimen portti Set-operaatiolla).
3. **Hälytyksiin (Traps):** Valvottava laite voi lähettää itsenäisesti välittömän hälytysviestin (Trap) valvontajärjestelmälle kriittisen virheen sattuessa (esim. virtalähteen rikkoutuessa).

---

## 2. Asennus

### Miten SNMP-agentti asennettiin?
Kurssiympäristössä SNMP-agentti (`snmpd`) asennettiin Ubuntu-pohjaisille Linux-palvelimille (`web1`, `db1` ja `branch-client`) kirjautumalla sisään kuhunkin konttiin vuorollaan ja ajamalla paketinhallinnan asennuskomento:

```bash
apt update && apt install snmpd -y
```

### Mitä konfiguraatiomuutoksia tehtiin?
Net-SNMP-agentin Ubuntun oletusasetukset tiedostossa `/etc/snmp/snmpd.conf` rajoittavat kuuntelun vain paikalliseen osoitteeseen (`127.0.0.1`) sekä näkymän pelkkiin suppeisiin järjestelmätietoihin (systemonly). Jotta valvontapalvelin (`ansible`) pystyy lukemaan laitteilta kaikki tarvittavat rajapintatiedot, asetustiedosto ylikirjoitettiin jokaisessa laitteessa yhdellä puhtaalla rivillä:

```bash
echo "rocommunity public default .1" > /etc/snmp/snmpd.conf
```
*Tämä määritys poistaa oletusrajoitukset, asettaa yhteisötunnukseksi `public` ja antaa sille lukuoikeudet koko MIB-puun juuresta (`.1`) alkaen.* Muutoksen jälkeen palvelu käynnistettiin uudelleen komennolla `service snmpd restart`.

---

## 3. Kerätyt tiedot

SNMP-kyselyt suoritettiin keskitetysti hallintakoneelta (`ansible`) käyttäen `snmpget`- ja `snmpwalk`-työkaluja. Koska kontissa ei ollut tekstimuotoisia MIB-kääntäjiä, kyselyt suoritettiin käyttämällä suoria numeerisia OID-osoitteita.

### Suoritetut SNMP-kyselyt ja komentotulosteet (`ansible`-kontista)

#### 1. Järjestelmän nimi (`sysName.0` -> `.1.3.6.1.2.1.1.5.0`)
```text
root@ansible:/# snmpget -v2c -c public web1 .1.3.6.1.2.1.1.5.0
iso.3.6.1.2.1.1.5.0 = STRING: "web1"
```

#### 2. Järjestelmän kuvaus (`sysDescr.0` -> `.1.3.6.1.2.1.1.1.0`)
```text
root@ansible:/# snmpget -v2c -c public web1 .1.3.6.1.2.1.1.1.0
iso.3.6.1.2.1.1.1.0 = STRING: "Linux web1 6.18.33.2-microsoft-standard-WSL2 #1 SMP PREEMPT_DYNAMIC Thu Jun 18 21:54:43 UTC 2026 x86_64"
```

#### 3. Järjestelmän käyttöaika (`sysUpTime.0` -> `.1.3.6.1.2.1.1.3.0`)
```text
root@ansible:/# snmpget -v2c -c public web1 .1.3.6.1.2.1.1.3.0
iso.3.6.1.2.1.1.3.0 = Timeticks: (619293) 1:43:12.93
```

### Usean laitteen valvonta (Tehtävä 2.7)
Kyselyt toistettiin onnistuneesti kaikille kolmelle kohdelaitteelle. Kerätyt tiedot on koottu alla olevaan taulukkoon:

| Laite | Nimi (sysName.0) | Käyttöjärjestelmä (sysDescr.0) | Uptime (sysUpTime.0) |
| :--- | :--- | :--- | :--- |
| **web1** | `web1` | Linux WSL2 (6.18.33.2-microsoft-standard) | 1 tunti, 43 minuuttia |
| **db1** | `db1` | Linux WSL2 (6.18.33.2-microsoft-standard) | 21 minuuttia, 58 sekuntia |
| **branch-client** | `branch-client` | Linux WSL2 (6.18.33.2-microsoft-standard) | 10 minuuttia, 15 sekuntia |

*Havainnot:* Kaikki laitteet jakavat saman WSL2-ytimen, koska ne pyörivät Docker-alustalla samalla isäntäkoneella. Uptimet eroavat toisistaan, sillä ne mittaavat kunkin kontin oman SNMPd-palveluprosessin pystyssäoloaikaa, joka nollautui asennuksen ja palvelun uudelleenkäynnistyksen yhteydessä.

---

## 4. Verkkorajapinnat

Laitteen `web1` verkkorajapinnat kartoitettiin `ansible`-kontista ajamalla `snmpwalk`-komento rajapintakuvauksille tarkoitetulla OID-oksalla `.1.3.6.1.2.1.2.2.1.2`:

```text
root@ansible:/# snmpwalk -v2c -c public web1 .1.3.6.1.2.1.2.2.1.2
iso.3.6.1.2.1.2.2.1.2.1 = STRING: "lo"
iso.3.6.1.2.1.2.2.1.2.112 = STRING: "eth0"
iso.3.6.1.2.1.2.2.1.2.119 = STRING: "eth1"
```

### Rajapintojen tunnistaminen ja tulkinta
Kyselyn perusteella `web1`-laitteesta tunnistettiin **kolme verkkorajapintaa**:
1. **`lo` (Index .1):** Virtuaalinen loopback-paikallisrajapinta sisäistä liikennettä varten.
2. **`eth0` (Index .112):** Containerlabin sisäinen hallintaportti, jonka kautta kontti kommunikoi suoraan hallintaverkon (kuten `172.20.20.x`) kanssa.
3. **`eth1` (Index .119):** Tuotantoverkon rajapinta, joka yhdistää `web1`-palvelimen Server LAN -aliverkkoon (`10.10.20.0/24`). Kaikki varsinainen asiakasliikenne ja reititys yhdyskäytävälle kulkevat tämän portin kautta.

---

## 5. OID-analyysi

SNMP-hallintatiedot perustuvat hierarkkiseen MIB-rakenteeseen, jossa jokaisella objektilla on numeerinen osoite eli **OID (Object Identifier)**. Harjoituksessa tutkitut keskeiset objektit ja niiden käyttötarkoitukset:

| OID / Objekti | Tarkoitus verkonvalvonnassa |
| :--- | :--- |
| **`.1.3.6.1.2.1.1.5.0` (sysName.0)** | **Laitteen nimi:** Palauttaa laitteen hostname-nimen. Valvontajärjestelmät (kuten Zabbix) käyttävät tätä nimeämään laitteen automaattisesti käyttöliittymässään. |
| **`.1.3.6.1.2.1.1.1.0` (sysDescr.0)** | **Järjestelmän kuvaus:** Kertoo laitteen valmistajan, käyttöjärjestelmän ja version. Auttaa valvontapalvelinta tunnistamaan automaattisesti laitteen tyypin. |
| **`.1.3.6.1.2.1.1.3.0` (sysUpTime.0)** | **Käyttöaika:** Kertoo sekunnin sadasosina ajan palvelun käynnistymisestä. Jos arvo tippuu yllättäen alas, valvonta tietää laitteen tai palvelun kaatuneen/käynnistyneen uudelleen. |
| **`.1.3.6.1.2.1.2.2.1.2` (ifDescr)** | **Rajapinnan kuvaus:** Palauttaa verkkoporttien nimet tekstinä (kuten `eth1`). Tämän avulla valvotut liikennemittarit (tavut sisään/ulos) voidaan kohdistaa oikeaan porttiin. |
| **`.1.3.6.1.2.1.2.2.1.8` (ifOperStatus)** | **Operatiivinen tila:** Kertoo onko portti linjoilla (`up (1)`) vai alhaalla (`down (2)`). Käytetään välittömien hälytysten luomiseen katkenneista kaapeleista. |

---

## 6. Pohdinta

### Mitä opit tehtävän aikana?
Tehtävän aikana opin SNMP-protokollan Manager-Agent-toimintaperiaatteen. Oli erittäin opettavaista törmätä käytännössä Net-SNMP:n oletusarvoisiin tietoturvarajoituksiin (systemonly ja localhost-lukitus) ja oppia muokkaamaan `snmpd.conf`-tiedostoa siten, että verkon yli tehtävät kyselyt ja rajapintojen selaus tulivat sallituiksi.

### Mitkä ovat SNMP:n tärkeimmät hyödyt?
* **Universaali standardi:** Lähes kaikki maailman verkkolaitteet (reitittimet, kytkimet, palvelimet, tulostimet) tukevat SNMP-valvontaa valmistajasta riippumatta.
* **Erittäin kevyt:** UDP-pohjaisena kyselyt ja vastaukset kuluttavat äärimmäisen vähän verkkokaistaa ja laitteen suorituskykyä.
* **Proaktiivisuus:** Mahdollistaa automaattiset hälytykset (Traps) ja vikojen havaitsemisen ennen kuin ne näkyvät loppukäyttäjille.

### Mitkä ovat SNMP:n rajoitukset tai haasteet?
* **Tietoturvan puute (v1 ja v2c):** Tässä harjoituksessa käytetty SNMPv2c lähettää yhteisötunnuksen (`public`) täysin salaamattomana (selkokielisenä tekstinä) verkon yli. Jos hyökkääjä pääsee käsiksi Management-liikenteeseen, hän voi napata tunnuksen ja urkkia laitteiden tietoja.
* **Suorituskykyriski laajoissa kyselyissä:** Jos suurelta kytkimeltä pyydetään kerralla koko rajapintapuuta (`snmpwalk`), se voi kuormittaa laitteen prosessoria ja aiheuttaa hetkellistä viivettä verkkoliikenteeseen.

### Missä tilanteissa käyttäisit mieluummin SNMPv3:a?
Käyttäisin SNMPv3-versiota **aina kaikissa tuotantoverkoissa**, yritysympäristöissä sekä julkisen internetin yli tapahtuvassa valvonnassa. SNMPv3 tuo mukanaan kryptografisen salauksen (liikenne ei ole luettavissa selkokielisenä) sekä käyttäjäkohtaisen tunnistautumisen (Authentication), mikä estää yhteisötunnusten urkinnan ja luvattomat muutokset laitteisiin. SNMPv2c-versiota voi käyttää luotettavasti ainoastaan täysin suljetuissa, eristetyissä laboratorio- tai hallintaverkoissa.