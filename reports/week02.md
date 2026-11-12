# Verkonhallinta: viikko 2

## 1 Johdanto

### 1.1 Mikä on SNMP?

SNMP on tietoliikenneprotokolla, jonka nimi on lyhenne sanoista *Simple Network Management Protocol*. Sen avulla voidaan kerätä tietoa laitteista, jotka sijaitsevat hallinnoidussa verkossa. Lisäksi sillä voidaan myös muokata ja valvoa kyseisten laitteiden toimintaa.

SNMP toimii Internet-protokollan sovelluskerroksessa. Sen avulla lähetetyt viestit välitetään UDP-protokollan avulla. Sen toiminta perustuu manager-agent-rakenteeseen, jossa agentti vastaa managerin kyselyihin. Kyselyiden ja vastausten oletusportti on 161 ja agentin lähettämien hälytysten oletusportti 162. Agentti asennetaan hallittavaan laitteeseen (esim. palvelin tai reititin). SNMP:n avulla voidaan mm. tehdä suorituskykymittauksia, joiden ansiosta häiriöitä voidaan havaita ja korjata proaktiivisesti.

SNMP:n avulla kerättävää tietoa luokitellaan MIB-tietokannan (*Management Information Base*) avulla. MIB:ssä jokainen muuttuja vastaa jotakin OID-tunnusta (*Object Identifier*). Näiden avulla voidaan kerätä mm. järjestelmätietoja, verkkorajapintojen tietoja tai suorituskykymetriikoita.

### 1.2 SNMPv2 VS. SNMPv3

SNMP:stä on olemassa useita eri versioita. Niiden suurimmat erot liittyvät tietoturvaan sekä pääsynhallintaan.

Verratessa SNMPv2:sta ja SNMPv3:sta, voidaan todeta SNMPv2:n olevan näistä tietoturvattomampi. Se on altis kuuntelulle, sillä dataa tai selkokielistä todentamistunnusta ei salata. SNMPv2 on kuitenkin laajasti tuettu ja sen konfigurointi on yksinkertaisempaa, minkä vuoksi sen käyttö on laajaa erityisesti suljetuissa ympäristöissä.

SNMPv3 puolestaan mahdollistaa tietoturvallisen järjestelmänhallinnan. Se tukee käyttäjätunnistautumista sekä datan salausta. Se myös tarkistaa viestien eheyden, toisin kuin SNMPv2, mikä edesauttaa monitorointia.

## 2 Asennus

Luvuissa 2.1-2.3 kuvatut muutokset tehtiin ```web1```, ```db1``` ja ```branch-client```-laitteisiin. Ansiblen muutokset on kuvattu luvussa 2.4.

### 2.1 SNMP-agentin asennus

1. Avattiin konfiguroitava laite
    - ```docker exec -it clab-hamk-verkonhallinta-golden-[laitteen_nimi] bash```

2. Asennettiin ```snmp``` ja ```snmpd```-paketit
    - ```apt install snmp snmpd -y```

3. Tehtiin tarvittavat konfiguraatiomuutokset (ks. luku 2.2)
    - ```nano /etc/snmp/snmpd.conf```

4. Käynnistettiin snmpd-agentti
    - ```service snmpd restart```

5. Tarkistettiin palvelun olevan käynnissä
    ```console
    service snmpd status
        * snmpd is running
    ps aux | grep snmpd
        Debian-+    4669  0.0  0.0  20236  7584 ?    S    12:41   0:00 /usr/sbin/snmpd -LSwd -Lf /dev/null -u Debian-snmp -g Debian-snmp -I -smux mteTrigger mteTriggerConf -p /run/snmpd.pid
    ```

### 2.2 Tehdyt konfiguraatiomuutokset

**snmpd.conf**

- Tarkistettiin, että Access Control Setupista löytyy *public readonly*-community
    - Alla oleva *rocommunity* löytyi jo tiedostosta, joten jätettiin se alkuperäiseen muotoonsa

    ```console
    # Read-only access to everyone to the systemonly view
    rocommunity  public default -V systemonly
    ```

- Lisäksi lisättiin Access Control Setupin Viewsien alle OID ```view   systemonly  included   .1.3.6.1.2```
    - Tämä kattaa koko MIB2-haaran, johon kuuluu sekä järjestelmä- että verkkorajapintatiedot

    ```console
    #  system + hrSystem groups only
    view   systemonly  included   .1.3.6.1.2.1.1
    view   systemonly  included   .1.3.6.1.2.1.25.1
    view   systemonly  included   .1.3.6.1.2
    ```

### 2.3 SNMP-agenttien sitominen hallintaverkkoon

Jotta saatiin laitteiden snmpwalk-komennot toimimaan Ansiblesta, piti niiden kuuntelu ensin sitoa hallintaverkon ip-osoitteeseen. Tämä tehtiin seuraavasti:

1. Tarkistettiin mikä ip-osoite ```eth0```-portille oli määritetty komennolla ```ip addr show eth0```
    - Oikea rajapinta saatiin selvitettyä ajamalla komento ```snmpwalk -v2c -c public localhost ifDescr``` kunkin laitteen rootissa

    **web1**

        ```console
        root@web1:/# ip addr show eth0
        26: eth0@if27: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
            link/ether 02:42:ac:14:14:09 brd ff:ff:ff:ff:ff:ff link-netnsid 0
            inet 172.20.20.9/24 brd 172.20.20.255 scope global eth0
            valid_lft forever preferred_lft forever
            inet6 fe80::42:acff:fe14:1409/64 scope link
            valid_lft forever preferred_lft forever
        ```
    **--> Käytettävä ip-osoite 172.20.20.9/24**

    ---

    **db1**

        ```console
        root@db1:/# ip addr show eth0
        24: eth0@if25: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
            link/ether 02:42:ac:14:14:08 brd ff:ff:ff:ff:ff:ff link-netnsid 0
            inet 172.20.20.8/24 brd 172.20.20.255 scope global eth0
            valid_lft forever preferred_lft forever
            inet6 fe80::42:acff:fe14:1408/64 scope link
            valid_lft forever preferred_lft forever
        ```
    **--> Käytettävä ip-osoite 172.20.20.8/24**

    ---

    **branch-client**

        ```console
        root@branch-client:/# ip addr show eth0
        16: eth0@if17: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
            link/ether 02:42:ac:14:14:05 brd ff:ff:ff:ff:ff:ff link-netnsid 0
            inet 172.20.20.5/24 brd 172.20.20.255 scope global eth0
            valid_lft forever preferred_lft forever
            inet6 fe80::42:acff:fe14:1405/64 scope link
            valid_lft forever preferred_lft forever
        ```
    **--> Käytettävä ip-osoite 172.20.20.5/24**

    ---

2. Päivitettiin laitteiden ```snmpd.conf```-tiedostoihin kyseiset ip-osoitteet ```agentaddress```-listaan. Kussakin laitteessa oli määriteltynä ```agentaddress  127.0.0.1,[::1]```, eli localhost ja IPv6-osoitteen kuuntelu. Nämä muokattiin seuraaviksi:
    - **web1:** ```agentaddress  127.0.0.1,[::1],172.20.20.9:161```
    - **db1:** ```agentaddress  127.0.0.1,[::1],172.20.20.8:161```
    - **branch-client:** ```agentaddress  127.0.0.1,[::1],172.20.20.5:161```

3. Tämän jälkeen ```snmpwalk -v2c -c public [laitteen_nimi] system```-komento toimi odotetusti  Ansiblesta

### 2.4 Ansible

1. Avattiin laite
    - ```docker exec -it clab-hamk-verkonhallinta-golden-ansible bash```

2. Asennettiin snmp-palvelu ja MIB-kirjasto
    - ```apt install -y snmp snmp-mibs-downloader```

3. Avattiin ```snmp.conf```-tiedosto
    - ```nano /etc/snmp/snmp.conf```

4. Enabloitiin MIBit kommentoimalla ```mibs :```-rivi pois snmp.conf-tiedostosta

    ```console
    # As the snmp packages come without MIB files due to license reasons, loading
    # of MIBs is disabled by default. If you added the MIBs you can reenable
    # loading them by commenting out the following line.
    # mibs :
    ```

## 3 Kerätyt tiedot

**Järjestelmän nimi**: ```snmpget -v2c -c public [laitteen_nimi] sysName.0```

```console
root@ansible:/# snmpget -v2c -c public web1 sysName.0
    SNMPv2-MIB::sysName.0 = STRING: web1
root@ansible:/# snmpget -v2c -c public db1 sysName.0
    SNMPv2-MIB::sysName.0 = STRING: db1
root@ansible:/# snmpget -v2c -c public branch-client sysName.0
    SNMPv2-MIB::sysName.0 = STRING: branch-client
```

---

**Käyttöjärjestelmä**: ```snmpget -v2c -c public [laitteen_nimi] sysDescr.0```

```console
root@ansible:/# snmpget -v2c -c public web1 sysDescr.0
    SNMPv2-MIB::sysDescr.0 = STRING: Linux web1 6.6.87.2-microsoft-standard-WSL2 #1 SMP PREEMPT_DYNAMIC Thu Jun  5 18:30:46 UTC 2025 x86_64
root@ansible:/# snmpget -v2c -c public db1 sysDescr.0
    SNMPv2-MIB::sysDescr.0 = STRING: Linux db1 6.6.87.2-microsoft-standard-WSL2 #1 SMP PREEMPT_DYNAMIC Thu Jun  5 18:30:46 UTC 2025 x86_64
root@ansible:/# snmpget -v2c -c public branch-client sysDescr.0
    SNMPv2-MIB::sysDescr.0 = STRING: Linux branch-client 6.6.87.2-microsoft-standard-WSL2 #1 SMP PREEMPT_DYNAMIC Thu Jun  5 18:30:46 UTC 2025 x86_64
```

---

**Uptime**: ```snmpget -v2c -c public [laitteen_nimi] sysUpTime.0```

```console
root@ansible:/# snmpget -v2c -c public web1 sysUpTime.0
    DISMAN-EVENT-MIB::sysUpTimeInstance = Timeticks: (667976) 1:51:19.76
root@ansible:/# snmpget -v2c -c public db1 sysUpTime.0
    DISMAN-EVENT-MIB::sysUpTimeInstance = Timeticks: (23476) 0:03:54.76
root@ansible:/# snmpget -v2c -c public branch-client sysUpTime.0
    DISMAN-EVENT-MIB::sysUpTimeInstance = Timeticks: (24611) 0:04:06.11
```

---

**Käynnissäolo**: ```snmpwalk -v2c -c public [laitteen_nimi] ifOperStatus```

```console
root@ansible:/# snmpwalk -v2c -c public web1 ifOperStatus
    IF-MIB::ifOperStatus.1 = INTEGER: up(1)
    IF-MIB::ifOperStatus.26 = INTEGER: up(1)
root@ansible:/# snmpwalk -v2c -c public db1 ifOperStatus
    IF-MIB::ifOperStatus.1 = INTEGER: up(1)
    IF-MIB::ifOperStatus.24 = INTEGER: up(1)
root@ansible:/# snmpwalk -v2c -c public branch-client ifOperStatus
    IF-MIB::ifOperStatus.1 = INTEGER: up(1)
    IF-MIB::ifOperStatus.16 = INTEGER: up(1)
```

---

### 3.1 Tulosteiden yhteenveto

| Laite | Nimi | Käyttöjärjestelmä | Uptime | Käynnissä |
|---------|---------|---------|---------|---------|
| web1 | web1 | Linux 6.6.87.2 (WSL2 / x86_64) | 1 h 51 min 19 s | Kyllä (1) |
| db1 | db1 | Linux 6.6.87.2 (WSL2 / x86_64) | 3 min 54 s | Kyllä (1) |
| branch-client | branch-client | Linux 6.6.87.2 (WSL2 / x86_64) | 4 min 6 s | Kyllä (1) |

Tästä voidaan havaita, että:
- Kaikki laitteet ajavat samanlaista WSL2-pohjaista Linux-ydintä
- Kaikkien laitteiden hallintaverkkoon yhteydessä oleva verkkorajapinta on aktiivisena
- ```web1```-laitteen uptime on huomattavasti pidempi, sillä kahden muun laitteen muutokset tehtiin myöhemmin, jolloin niiden agentit ovat olleet käynnissä huomattavasti vähemmän aikaa

## 4 Verkkorajapinnat

Käytiin läpi laitteiden verkkorajapinnat seuraavalla komennolla: ```snmpwalk -v2c -c public [laitteen_nimi] ifDescr```

Kultakin laitteelta löytyi kaksi kappaletta verkkorajapintoja: *lo* ja *eth0*. lo on loopback-rajapinta, kun taas eth0 on yhteydessä Containerlabin ```172.20.20.0```-hallintaverkkoon.

**web1**

```console
root@ansible:/# snmpwalk -v2c -c public web1 ifDescr
    IF-MIB::ifDescr.1 = STRING: lo
    IF-MIB::ifDescr.26 = STRING: eth0
```

**db1**

```console
root@ansible:/# snmpwalk -v2c -c public db1 ifDescr
    IF-MIB::ifDescr.1 = STRING: lo
    IF-MIB::ifDescr.24 = STRING: eth0
```

**branch-client**

```console
root@ansible:/# snmpwalk -v2c -c public branch-client ifDescr
    IF-MIB::ifDescr.1 = STRING: lo
    IF-MIB::ifDescr.16 = STRING: eth0
```

## 5 OID-analyysi

| OID | Tarkoitus |
|------|------|
| sysName.0 | Kertoo laitteen hallinnollisen nimen. Vakiintunut käytäntö on nimetä se toimialueen nimen mukaan. |
| sysDescr.0 | Tekstimuotoinen kuvaus laitteesta. Sisältää tiedon laitteistotyypistä, käyttöjärjestelmästä, sekä verkko-ohjelmistosta. |
| sysUpTime.0 | Sadasosasekunteina määritelty aika siitä, kun järjestelmä on viimeksi käynnistynyt uudelleen. |
| ifDescr | Tekstimuotoinen kuvaus rajapinnasta. Kertoo valmistajan nimen, tuotenimen sekä rajapinnan laitteiston/ohjelmiston version. |
| ifOperStatus | Kertoo rajapinnan tämänhetkisen operatiivisen tilan kokonaisluvulla: 1 (käynnissä), 2 (alhaalla), 3 (testaus), 4 (tuntematon), 5 (lepotila), 6 (ei läsnä), tai 7 (alempi kerros alhaalla).  |

## 6 Pohdinta

Tehtävän aikana opin SNMP:stä ja sen toimintaperiaatteista. Opin mm. kuinka asentaa ja konfiguroida valvottavia laitteita niin, että ne osaavat vastata hallintaverkon kyselyihin. Lisäksi tuntui, että sain parempaa rutiinia Linuxin komentorivipohjaiseen käyttöön.

Mielestäni SNMP:n tärkeimpiä hyötyjä on sen laaja tuki (toimii käytännössä lähes missä vaan laitteessa valmistajasta ja mallista riippumatta). Ymmärsin kuitenkin myös minkä vuoksi tehtävässä käytettyä SNMPv2:ta ei tulisi käyttää kuin eristetyissä ympäristöissä - jos kyseessä olisi tuotantoympäristö, olisi SNMPv3 huomattavasti tietoturvallisempi vaihtoehto, sillä SNMPv2 ei salaa liikennettä, tue käyttäjätodennusta tai tarkista lähetettyjen viestien eheyttä toisin kuin SNMPv3.