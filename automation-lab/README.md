# Korvaava laboratorio: Verkon automaatio Ansiblella

## 1. Ympäristön tarkistaminen
Ympäristön tila tarkistettiin komennoilla `sudo containerlab inspect --all` ja `docker ps`. Kaikki tarvittavat kontit olivat käynnissä Containerlabin hallintaverkossa (`172.20.20.0/24`):
* **Ansible-hallintapalvelin (`clab-hamk-verkonhallinta-golden-ansible`):** `172.20.20.3`
* **r1-edge (`clab-hamk-verkonhallinta-golden-r1`):** `172.20.20.17`
* **r2-core (`clab-hamk-verkonhallinta-golden-r2`):** `172.20.20.5`
* **r3-branch (`clab-hamk-verkonhallinta-golden-r3`):** `172.20.20.10`

**Miten Ansible-palvelin kommunikoi reitittimien kanssa?**
Ansible-palvelin ja FRRouting-reitittimet sijaitsevat samassa hallintaverkossa (`172.20.20.0/24`). Ansible ottaa yhteyden reitittimiin SSH-protokollan yli avainparilla ja suorittaa tehtävät Python-tulkin avulla ilman erillistä agenttia.

## 2. SSH-yhteydet ja Ansible-inventaario
Asensin reitittimille OpenSSH-palvelimen ja Python 3:n sekä kopioin julkisen SSH-avaimen reitittimille. SSH-yhteys toimi kaikkiin kolmeen reitittimeen ilman salasanaa (`r1`, `r2`, `r3`), ja `ansible -i inventory/hosts.yml routers -m ping` palautti kaikilta `SUCCESS / pong`.

### Inventaario (`ansible-inventory -i inventory/hosts.yml --graph`)
```text
@all:
  |--@ungrouped:
  |--@routers:

  |  |--r1-edge
  |  |--r2-core
  |  |--r3-branch
```

## 3. Verkon automaattinen inventointi (`playbooks/facts.yml`)

| Ominaisuus | r1-edge (`r1`) | r2-core (`r2`) | r3-branch (`r3`) |
| :--- | :--- | :--- | :--- |
| **Hallinta-IP (`eth0`)** | `172.20.20.17/24` | `172.20.20.5/24` | `172.20.20.10/24` |
| **Käyttöjärjestelmä** | Alpine Linux 3.16.2 | Alpine Linux 3.16.2 | Alpine Linux 3.16.2 |
| **Uptime** | `66265 s` | `66265 s` | `66265 s` |
| **Verkkointerfacet** | `lo`, `eth0`, `eth1`, `eth2` | `lo`, `eth0`, `eth1`, `eth2` | `lo`, `eth0`, `eth1`, `eth2` |

**Kysymys: Mitä eroja havaitset reitittimien interface- ja IP-konfiguraatioissa?**
Jokaisella reitittimellä on oma hallinta-IPv4-osoite `172.20.20.0/24`-verkossa (`.17`, `.5`, `.10`), oma MAC- ja link-local-osoitteensa sekä oma roolinsa FRRoutingin OSPF-topologiassa (router-id `1.1.1.1`, `2.2.2.2`, `3.3.3.3` ja haarakonttorin verkko `10.10.30.0/24`).

## 4. Konfiguraatioiden varmistaminen (`playbooks/backup.yml`)
Playbook hakee `vtysh -c "show running-config"` -tulosteen ja tallentaa varmuuskopiot kansioon `backups/` (`r1-edge.conf`, `r2-core.conf`, `r3-branch.conf`).

## 5. Verkon tila Git-versionhallintaan
Alustin harjoitushakemiston Git-repositorioksi (`git init`) ja tallensin alkutilan commitilla `Initial network state` (`d99a1f7`).
* **Miksi konfiguraatioita kannattaa säilyttää versionhallinnassa?** Versionhallinta tarjoaa tarkan muutoshistorian (`git log`, `git diff`), jäljitettävyyden ja mahdollisuuden palata nopeasti tunnettuun toimivaan tilaan virheen sattuessa.

## 6. Hallittu konfiguraatiomuutos (`playbooks/loopback.yml`)
Lisäsin `r3-branch`-reitittimelle loopback-interfacen `lo10` osoitteella `192.0.2.3/32`.
* **1. suoritus:** `ok=2, changed=1, failed=0` (muutos toteutui).
* **2. suoritus:** `ok=1, changed=0, failed=0, skipped=1` (ei muutoksia).
* **Mitä tulos kertoo Ansiblen idempotenssista?** Playbookin voi ajaa turvallisesti uudelleen ilman sivuvaikutuksia: kun tavoitetila on jo voimassa, Ansible ei tee uusia muutoksia (`changed=0`).

## 7. Muutoksen tarkistaminen (`playbooks/verify.yml`)
Playbook tarkistaa `ping`- ja `assert`-moduuleilla, että kaikki reitittimet vastaavat ja `lo10`-interfacella on osoite `192.0.2.3/32`.
* **Miksi muutoksen tekeminen ja sen onnistumisen tarkistaminen ovat automaatiossa eri asioita?** Komennon ajaminen ilman virhettä ei vielä takaa, että liitäntä tai reititys toimii oikein. Erillinen tarkistus varmistaa halutun lopputilan ennen jatkotoimia.

## 8. Muutoksen tallentaminen
Päivitin varmuuskopiot, tarkistin muutoksen `git diff` -komennolla ja tallensin tilan commitilla `Add branch loopback interface` (`de56946`).

## 9. Virhetilanne ja palautuminen
Simuloin virheen vaihtamalla `lo10`-osoitteeksi `192.0.2.99/32`. Uusi backup ja `git diff` paljastivat muuttuneen IP-osoitteen, ja `verify.yml` epäonnistui `assert`-tarkistuksessa. Poistin virheellisen osoitteen ja ajoin `loopback.yml`-playbookin, joka palautti oikean osoitteen `192.0.2.3/32`. Lopuksi `verify.yml` vahvisti verkon toimivan tilan.

## Yhteenveto ja vastaukset loppukysymyksiin
* **Ansiblen hyödyt verrattuna käsin konfigurointiin:** Poistaa inhimilliset näppäilyvirheet, nopeuttaa useiden laitteiden hallintaa, takaa yhdenmukaiset asetukset eikä vaadi agentteja kohdelaitteille.
* **Ansiblen idempotenssi käytännössä:** Toistuvat ajot tuottavat aina saman lopputuloksen eivätkä muuta jo oikein konfiguroituja kohteita (`changed=0`).
* **Git-versionhallinnan hyödyt:** Muutosten auditointi, vertailu (`git diff`) ja nopea palautuminen (rollback) virhetilanteissa.
* **Virtuaalisten FRRouting-reitittimien ero oikeisiin verkkolaitteisiin:** FRR-kontit pyörivät Linuxin päällä
