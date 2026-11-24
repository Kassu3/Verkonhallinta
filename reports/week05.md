# Viikko 5 – Tietoturva, lokit ja verkkoliikenteen analysointi

## Johdanto
Verkonhallinnassa tietoturva, lokien seuranta ja verkkoliikenteen analysointi ovat olennaisia keinoja havaita poikkeavaa liikennettä, luvattomia tiedusteluyrityksiä (kuten porttiskannauksia), väärinkäytöksiä sekä virheellisiä järjestelmäkonfiguraatioita. Pakettikaappauksen ja palvelinlokien yhdistäminen mahdollistaa tapahtumien tarkan aikajanan ja juurisyiden selvittämisen.

---

## Kaappaus
Liikenteen kaappaus suoritettiin Containerlab-ympäristön `web1`-kontissa (`clab-hamk-verkonhallinta-golden-web1`).

1. Ennen kaappauksen aloitusta `web1`-kontin verkkoliittymät tarkistettiin komennolla `ip a`. Havaittiin, että tehtävänannossa mainittua `eth1`-liittymää (`10.10.20.101`) ei ollut olemassa, vaan kontilla oli käytössä hallintaliittymä **`eth0`** IP-osoitteella **`172.20.20.6/24`**.
2. Pakettikaappaus käynnistettiin `web1`-kontissa komennolla:
   ```bash
   tcpdump -i eth0 -w /tmp/web1.pcap
   ```
3. Kaappauksen ollessa käynnissä `attacker`-kontista (`172.20.20.13`) ajettiin Nmap-porttiskannaus sekä testi-HTTP- ja DNS-kyselyt kohteeseen `172.20.20.6`.
4. Kaappaus pysäytettiin (`Ctrl + C`) ja `web1.pcap`-tiedosto siirrettiin työasemalle komennolla `docker cp clab-hamk-verkonhallinta-golden-web1:/tmp/web1.pcap .` Wireshark-analyysiä varten.

---

## Porttiskannaus
`attacker`-kontissa ajettiin komento `nmap 172.20.20.6`, joka suoritti TCP SYN -skannauksen 1000 yleisimpään TCP-porttiin 0,20 sekunnissa.

* **Skannauksen lähde (`attacker` IP):** `172.20.20.13`
* **Skannauksen kohde (`web1` IP):** `172.20.20.6` (MAC: `02:42:ac:14:14:06`)
* **Testatut portit ja palvelimen vastaukset:** Nmap testasi 1000 yleisintä TCP-porttia lähettämällä nopeassa tahdissa TCP `[SYN]` -paketteja. Koska `web1`-kontissa ei ollut käynnissä kuuntelevia verkkopalveluita, kaikki 1000 porttia olivat suljettuja (`Not shown: 1000 closed tcp ports (reset)`), ja `web1` vastasi jokaiseen yhteydenottoyritykseen välittömästi **`[RST, ACK]`** -paketilla.

![Porttiskannaus Wiresharkissa](kuva1_porttiskannaus.png)

---

## Wireshark (DNS- ja HTTP-liikenne)

### DNS-liikenne (Suodatin: `dns`)
* **DNS-kysely:** `attacker` (`172.20.20.13`) lähetti UDP-porttiin `53` A-tietuekyselyn verkkotunnukselle `hamk.fi` (`Standard query A hamk.fi`) suoraan `web1`-palvelimelle (`172.20.20.6`).
* **DNS-vastaus:** Koska `web1`-kontissa ei toimi DNS-palvelinta, palvelin vastasi välittömästi ICMP-virheilmoituksella **`Destination unreachable (Port unreachable)`** (type 3, code 3).

### HTTP-liikenne (Suodattimet: `http` ja `tcp.port == 80`)
* **HTTP-pyyntö:** Suodattimella `http` ei näkynyt varsinaisia HTTP-sovelluskerroksen paketteja (`GET / HTTP/1.1`), koska TCP-kolmikättely ei koskaan valmistunut. Suodattimella `tcp.port == 80` havaittiin `curl -v http://172.20.20.6` -komennon lähettämä TCP `[SYN]` -paketti lähdeportista `46898` kohdeporttiin `80`.
* **HTTP-vastaus:** `web1`-palvelimella ei ollut web-palvelinprosessia (Nginx/Apache) käynnissä, joten se hylkäsi yhteyden suoraan kuljetuskerroksella **`[RST, ACK]`** -paketilla (`Connection refused`).

![DNS-kyselyt ja ICMP-vastaukset Wiresharkissa](kuva2_dns.png)

---

## Loki-analyysi
`web1`-kontin `/var/log/`-hakemiston sisältö ja käynnissä olevat prosessit (`ps aux`) tarkastettiin.

* **Järjestelmä- ja kirjautumislokit:** Kontissa on käynnissä vain komentotulkki (`/bin/bash`) eikä `rsyslog`- tai `systemd`-palveluita ole aktiivisena, joten perinteistä `/var/log/syslog`-tiedostoa ei ole. Kirjautumislokit (`lastlog`, `wtmp`, `btmp`, `faillog`) olivat kooltaan `0` tavua, koska ylläpitotoimet on suoritettu suoraan Docker-isännältä `docker exec` -komennolla, joka ei kirjaudu perinteisiin TTY/SSH-kirjautumislokeihin.
* **Pakettienhallinnan loki (`/var/log/dpkg.log`):** Lokista varmistettiin kontin rakennusvaiheessa asennetut ja päivitetyt paketit (esim. `python3-launchpadlib`, `ssh-import-id` ja `libc-bin` aikaleimalla `2026-11-24 01:59:05`).
* **Sovellusloki (`/var/log/node_exporter.log`):** Lokista havaittiin Prometheus Node Exporterin (v1.8.2) käynnistys ja toiminta aikaleimalla `2026-11-24T01:59:10Z`:
  * **Palvelun käynnistys (`level=info`):** `Starting node_exporter` ja kuunteluosoitteen avaus (`Listening on address=[::]:9100`).
  * **Virhetilanne (`level=error`):** `Failed to open directory, disabling udev device properties path=/run/udev/data` (konttiympäristöstä puuttuva `udev`-hakemisto).
  * **Tietoturvavaroitukset (`level=warn` / `level=info`):** Lokissa oli suora varoitus pääkäyttäjäoikeuksista (`Node Exporter is running as root user. This exporter is designed to run as unprivileged user, root is not required.`) sekä ilmoitus salaamattomasta yhteydestä (`TLS is disabled. http2=false address=[::]:9100`).

---

## Turvallisuusarvio ja pohdinta

### Ympäristön turvallisuusarvio
1. **Mikä on turvallista:**
   * **Minimaalinen hyökkäyspinta-ala:** `web1`-kontissa ei ole turhia taustapalveluita tai avoimia verkkoportteja päällä, jolloin ulkopuolinen hyökkääjä ei pääse hyödyntämään palveluhaavoittuvuuksia verkon yli.
   * **Konttieristys:** Palvelu on eristetty omaan konttiinsa, jolloin yksittäisen palvelun vaarantuminen ei suoraan anna pääsyä isäntäkoneen resursseihin.
2. **Tietoturvariskit:**
   * **Palveluiden ajaminen `root`-oikeuksilla:** `node_exporter.log` osoitti palvelun käynnistyneen `root`-käyttäjänä. Jos palvelussa olisi haavoittuvuus, hyökkääjä saisi suoraan pääkäyttäjän oikeudet konttiin.
   * **Salaamaton liikenne ja puuttuva autentikointi:** `node_exporter` kuunteli porttia `9100` ilman TLS-salausta (`TLS is disabled`), jolloin järjestelmätiedot ovat kaikkien samassa verkossa olevien luettavissa selkokielisenä.
   * **Palomuurin ja pääsynhallinnan puute:** `attacker`-kontti pystyi skannaamaan kaikki 1000 porttia esteettä ja sai suorat `RST, ACK` -vastaukset ilman, että mikään palomuuri tai IDS/IPS-järjestelmä olisi hidastanut tai estänyt skannausta.
   * **Puutteellinen lokitus:** Kontin sisäiset lokit katoavat kontin uudelleenkäynnistyksessä, eikä `docker exec` -istuntoja tallennu lokiin.
3. **Parannusehdotukset:**
   * Ajetaan sovellukset ja exporter-palvelut vähimpien oikeuksien periaatteella (erillinen ei-etuoikeutettu käyttäjätunnus `root`-käyttäjän sijaan).
   * Otetaan käyttöön TLS-salaus ja autentikointi (esim. käänteisvälityspalvelimen tai Node Exporterin `web.config.yml`-asetusten avulla).
   * Keskitetään lokit (esim. laboratorion `syslog`-konttiin `clab-hamk-verkonhallinta-golden-syslog` tai Dockerin lokiajureilla), jotta lokit säilyvät ja hyökkäykset havaitaan.
   * Tiukennetaan verkkosegmentointia ja palomuurisääntöjä (esim. `iptables` / `nftables`), jotta vain valvontapalvelin (Prometheus) saa ottaa yhteyden porttiin `9100`.

### Pohdinta
* **Miten porttiskannaus näkyi liikenteessä?** Se näkyi selkeänä piikkinä liikennemäärässä: yksittäinen lähde-IP (`172.20.20.13`) lähetti satoja TCP `[SYN]` -paketteja peräkkäin eri kohdeportteihin murto-osasekunnissa, ja kohde vastasi jokaiseen `[RST, ACK]` -paketilla.
* **Miten se näkyi lokeissa?** `web1`-kontin sovelluslokeissa skannaus ei näkynyt lainkaan, koska yhtäkään sovelluskerroksen yhteyttä ei muodostunut eikä palomuurin pakettihylkäyksiä (esim. `iptables LOG`) ollut määritetty.
* **Mitä hyötyä verkkoliikenteen analysoinnista on?** Pakettitason analyysi paljastaa myös sellaiset tiedustelu- ja hyökkäystoimet (kuten SYN-skannaukset, salaamattomat salasanat ja virheelliset reititykset), jotka eivät koskaan päädy sovelluskerroksen lokeihin.
* **Miten monitorointia voisi hyödyntää hyökkäysten tunnistamisessa?** Yhdistämällä keskitetty lokienkeruu (Syslog/SIEM), verkkoliikenteen virta-analyysi (NetFlow/IDS kuten Suricata) ja metrikoiden seuranta (Prometheus/Zabbix) voidaan luoda automaattisia hälytyksiä esimerkiksi äkillisestä TCP SYN- tai RST-pakettien määrän kasvusta tai epäonnistuneista kirjautumisyrityksistä.

---

## Yhteenveto
Harjoituksessa opittiin kaappaamaan verkkoliikennettä komentoriviltä `tcpdump`-työkalulla, siirtämään kaappaustiedosto työasemalle ja analysoimaan TCP-, DNS- ja HTTP-liikennettä Wiresharkissa. Lisäksi harjoiteltiin konttiympäristön verkkotopologian ja lokitiedostojen tutkimista sekä yhdistettiin liikenne- ja lokihavainnot käytännön tietoturva-arvioksi.
