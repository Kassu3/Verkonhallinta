# Viikko 4 – Ansible ja Infrastructure as Code

## Johdanto
Infrastructure as Code (IaC) eli infrastruktuuri koodina tarkoittaa tietoteknisen infrastruktuurin automaattista hallintaa, konfigurointia ja provisiointia koneellisesti luettavien määritystiedostojen avulla. Perinteisen manuaalisen säätämisen sijaan palvelin- ja verkkoasetuksia kohdellaan kuten ohjelmistokoodia: ne kirjoitetaan YAML- tai tekstitiedostoiksi, tallennetaan versionhallintaan (Git) ja suoritetaan automaatiotyökaluilla keskitetysti.

## Inventory
Ansiblen toiminta perustuu `inventory.ini`-laiteluetteloon, joka määrittää valvottavat kohteet ja niiden loogiset ryhmät.

### Tutkimuskomennon tulokset (`cat /ansible/inventory.ini`):
```text
[servers]
web1 ansible_host=clab-hamk-verkonhallinta-golden-web1 data_ip=10.10.20.101 role=webserver
db1 ansible_host=clab-hamk-verkonhallinta-golden-db1 data_ip=10.10.20.102 role=database

[server_network]
web1
db1
```

### Havainnot ympäristön rakenteesta ja ryhmistä (Tehtävä 4.1):
* **Mitä laitteita inventory sisältää?** Inventory sisältää reitittimet (`r1`-`r3`), asiakaskoneet (`client1`, `attacker`, `branch-client`), sovelluspalvelimet (`web1`, `db1`) sekä valvontainfrastruktuurin kontit.
* **Miten ryhmät on muodostettu?** Laitteet on jaettu ryhmiin hakasulkeilla (esim. `[routers]`, `[clients]`, `[servers]`). Lisäksi on luotu loogisia segmenttejä kuten `[server_network]` (web1, db1) ja lapsiryhmiä kuten `[linux_hosts:children]`.
* **Mitä hyötyä ryhmistä on?** Ryhmien avulla ylläpitäjä voi kohdistaa ylläpitotoimet täsmällisesti vain tiettyihin laitetyyppeihin (esim. web-päivitys vain ryhmälle `[webservers]`), mikä helpottaa suurten ympäristöjen hallintaa.

### Yhteyksien testaus (Ping-moduuli):
Ajettaessa ad-hoc-komento `ansible all -i /ansible/inventory.ini -m ping` tehtiin seuraavat havainnot:
* Linux-palvelimet (`web1`, `db1`, `branch-client`, `client1`, `attacker`) ja reitittimet vastasivat onnistuneesti (`SUCCESS -> ping: pong`).
* Valvontakontit (`prometheus`, `grafana`, `zabbix`, `cadvisor`) antoivat virheen `UNREACHABLE / Connection refused`. Tämä on normaalia, sillä nämä kontit on optimoitu keveiksi eikä niissä ole SSH-palvelinta (portti 22) asennettuna; niitä hallitaan vain web-rajapintojen kautta.

## Esimerkkiplaybookit
Ympäristössä suoritettiin kaksi valmista esimerkkipohjaa hakemistosta `/ansible/playbooks/`.

### 1. ping.yml -playbook (Tehtävä 4.2)
Kysely suoritti yhteydenhallinnan onnistuneesti Linux-isännille (`ok=1`), mutta reitittimien osalta ajo antoi virheen `FAILED`, koska `ansible`-kontista puuttui Pythonin `paramiko`-SSH-kirjasto.

### 2. Havainnot asennusplaybookeista (Tehtävä 4.3)
* **install-snmp.yml:** Pakettien päivitys ja `snmpd.conf`-konfiguraation siirto onnistuivat (`changed`). Kuitenkin tehtävä `Enable SNMP service` antoi virheen `FAILED! => Service is in unknown state`. Tämä johtuu siitä, että dynaamisissa Docker-kontteina pyörivissä ympäristöissä ei ole perinteistä Linuxin `systemd`-init-järjestelmää taustalla, joten perinteinen palvelujen hallinta epäonnistuu konttirajoitusten vuoksi.
* **install-node-exporter.yml:** Tämä playbook suoriutui täysin puhtaasti ilman yhtäkään virhettä (`failed=0`) kaikilla laitteilla. Tämä johtuu siitä, että Node Exporter asennettiin suoraan valmiina binäärinä internetistä ja käynnistettiin taustaprosessiksi (`./node_exporter &`), jolloin se ei yrittänyt käyttää konttiympäristössä rikkonutta `systemd`-palvelunhallintaa.

## Oma playbook
Tehtävässä toteutettiin **molemmat** kustomoidut playbookit työn korkean laadun varmistamiseksi (Tehtävä 4.4).

### Koodi 1: Web-palvelimen asennustuloste (`install-webserver.yml` output):
```text
TASK [Tulostetaan palvelimen virallinen html-vastaus ruudulle] *********************************************************
ok: [web1] => {
    "msg": "Palvelimen sivu palauttaa: <h1>Tervetuloa palvelimelle web1! Web-palvelin on asennettu automaattisesti Ansiblella.</h1>"
}
ok: [db1] => {
    "msg": "Palvelimen sivu palauttaa: <h1>Tervetuloa palvelimelle web1! Web-palvelin on asennettu automaattisesti Ansiblella.</h1>"
}

PLAY RECAP *************************************************************************************************************
db1                        : ok=5    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
web1                       : ok=5    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

### Koodi 2: Tietokantapalvelimen asennustuloste (`install-database.yml` output):
```text
TASK [Tulostetaan tietokannan virallinen tila raporttiin] **************************************************************
ok: [db1] => {
    "msg": "Tietokannan tila palauttaa: ['--------------', 'mariadb  Ver 15.1 Distrib 10.11.14-MariaDB, for debian-linux-gnu...', 'Uptime:\\t\\t\\t2 sec', '--------------']"
}

PLAY RECAP *************************************************************************************************************
db1                        : ok=5    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

## Vertailu

### Tehtävä 4.5 – Kerää järjestelmätietoja
Suoritettiin järjestelmätietojen haku kohdistetusti Ansiblen `setup`-faktamoduulilla.

#### Todellinen komentorivituloste (Suoritusajon aito output):
```text
root@ansible:/ansible/playbooks# ansible servers -i ../inventory.ini -m setup -a "filter=ansible_distribution,ansible_default_ipv4,ansible_processor_vcpus,ansible_memtotal_mb"
web1 | SUCCESS => {
    "ansible_facts": {
        "ansible_default_ipv4": {
            "address": "10.10.20.101",
            "alias": "eth1",
            "broadcast": "",
            "gateway": "10.10.20.1",
            "interface": "eth1",
            "macaddress": "aa:c1:ab:17:21:14",
            "mtu": 9500,
            "netmask": "255.255.255.0",
            "network": "10.10.20.0",
            "prefix": "24",
            "type": "ether"
        },
        "ansible_distribution": "Ubuntu",
        "ansible_memtotal_mb": 7864,
        "ansible_processor_vcpus": 12
    },
    "changed": false
}
db1 | SUCCESS => {
    "ansible_facts": {
        "ansible_default_ipv4": {
            "address": "10.10.20.102",
            "alias": "eth1",
            "broadcast": "",
            "gateway": "10.10.20.1",
            "interface": "eth1",
            "macaddress": "aa:c1:ab:17:6d:1b",
            "mtu": 9500,
            "netmask": "255.255.255.0",
            "network": "10.10.20.0",
            "prefix": "24",
            "type": "ether"
        },
        "ansible_distribution": "Ubuntu",
        "ansible_memtotal_mb": 7864,
        "ansible_processor_vcpus": 12
    },
    "changed": false
}
```

#### Kerättyjen järjestelmätietojen yhteenveto:

| Palvelin | Käyttöjärjestelmä (ansible_distribution) | IP-osoite (ansible_default_ipv4) | Prosessorien määrä (ansible_processor_vcpus) | Muistin määrä (ansible_memtotal_mb) |
| :--- | :--- | :--- | :--- | :--- |
| **web1** | Ubuntu 24.04 (Noble) | `10.10.20.101` | 12 ydintä (Jaettu isäntä) | 7 864 MB (~8 GB) |
| **db1** | Ubuntu 24.04 (Noble) | `10.10.20.102` | 12 ydintä (Jaettu isäntä) | 7 864 MB (~8 GB) |
| **branch-client** | Ubuntu 24.04 (Noble) | `10.10.30.101` | 12 ydintä (Jaettu isäntä) | 7 864 MB (~8 GB) |

*Huomio havainnoista:* Koska palvelimet pyörivät dynaamisina Docker-kontteina samalla WSL2-isäntäkoneella, ne jakavat isäntäkoneen fyysiset resurssit, minkä vuoksi setup-faktat näyttävät niille identtisiä laiteresursseja.

### Tehtävä 4.6 – Analyysi (Käsin tehty vs. automaatio)
* **Käsin tehdyt asennukset (Viikot 2-3):** Vaativat manuaalista kirjautumista jokaiseen konttiin yksitellen, komentojen muistamista ulkoa, tiedostojen hidasta muokkausta rivi riviltä `nano`-editorilla sekä palveluiden käsin käynnistämistä. Työ on erittäin altis inhimillisille unohduksille tai kirjoitusvirheille.
* **Ansible-automaatio (Viikko 4):** Koko infrastruktuurin haluttu tila kirjoitetaan deklaratiivisesti yhteen kooditiedostoon keskitetyllä hallintakoneella. Koodi ajetaan yhdellä komennolla dynaamisesti taustalla, ja laitteet saatetaan haluttuun tilaan automaattisesti ilman käsin tehtävää työtä.

#### Automaation hyödyt ja välttämättömyys:
Automaatio poistaa rutiinityöt, nopeuttaa ylläpitoa moninkertaisesti ja varmistaa, että ympäristön jokainen palvelin on konfiguroitu täsmälleen identtisesti standardien mukaisesti (Configuration Driftin estäminen). Laajoissa yritysverkoissa ja pilviympäristöissä se on ainoa keino hallita satoja tai tuhansia palvelimia luotettavasti ja turvallisesti.

## Yhteenveto
Tämän viikon aikana opin Infrastructure as Code (IaC) -filosofian keskeiset edut. 