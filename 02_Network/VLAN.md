# Plan de segmentation et d'adressage — AEROSEC INDUSTRIES

## 1. Table des segments

| ID | Nom | Sous-réseau | Passerelle (pfSense) | Niveau Purdue | Interface pfSense | Réseau interne VirtualBox |
|---|---|---|---|---|---|---|
| — | WAN | DHCP NAT | — | 5 | `em0` | NAT |
| VLAN 10 | ADM-SOC | 192.168.10.0/24 | 192.168.10.1 | Transverse | `em1` | `aero-adm` |
| VLAN 20 | USERS | 192.168.20.0/24 | 192.168.20.1 | 4 | `em2` | `aero-users` |
| VLAN 30 | SERVERS | 192.168.30.0/24 | 192.168.30.1 | 4 | `em3` | `aero-srv` |
| VLAN 50 | DMZ | 192.168.50.0/24 | 192.168.50.1 | 4 (exposé) | `em4` | `aero-dmz` |
| VLAN 55 | IDMZ | 192.168.55.0/24 | 192.168.55.1 | **3.5** | `em5` | `aero-idmz` |
| VLAN 60 | OT-SUP | 192.168.60.0/24 | 192.168.60.1 | 3 / 2 | `em6` | `aero-otsup` |
| VLAN 65 | OT-PROC | 192.168.65.0/24 | 192.168.65.1 | 1 / 0 | `em7` | `aero-otproc` |

> Les VLAN 40 et 70 du cahier des charges initial sont fusionnés dans le VLAN 10
> (limite VirtualBox : 8 interfaces réseau maximum par VM). Écart documenté
> dans `01_Architecture/Description.md` §9.

## 2. Attribution des adresses

| Plage | Usage |
|---|---|
| `.1` | Passerelle pfSense |
| `.10 – .49` | Serveurs et équipements à IP fixe |
| `.100 – .199` | Baux DHCP (VLAN 20 uniquement) |
| `.200 – .254` | Réservé (tests, VM temporaires) |

### Adresses attribuées

| Machine | Segment | IP | Notes |
|---|---|---|---|
| FW-AERO-01 | toutes | `.1` de chaque segment | pfSense |
| SOC-SIEM-01 | VLAN 10 | 192.168.10.10 | Wazuh — ports 1514/1515/55000/443 |
| SRV-BKP-01 | VLAN 10 | 192.168.10.20 | Dépôt de sauvegarde (mode *pull*) |
| WKS-ADM-01 | VLAN 10 | 192.168.10.30 | Poste d'administration (PAW) |
| WKS-IT-01 | VLAN 20 | DHCP (192.168.20.100+) | Poste bureautique |
| KALI-RT-01 | VLAN 20 | 192.168.20.200 | Machine de test — éteinte hors scénario |
| SRV-DC-01 | VLAN 30 | 192.168.30.10 | AD DS, DNS, DHCP |
| SRV-FS-01 | VLAN 30 | 192.168.30.20 | Serveur de fichiers |
| DMZ-WEB-01 | VLAN 50 | 192.168.50.10 | Nginx / reverse proxy |
| IDMZ-JUMP-01 | VLAN 55 | 192.168.55.10 | Bastion + historian miroir |
| OT-SCADA-01 | VLAN 60 | 192.168.60.10 | ScadaBR / Node-RED + IHM |
| WKS-ENG-01 | VLAN 60 | 192.168.60.20 | Station d'ingénierie (hors domaine) |
| OT-PLC-01 | VLAN 65 | 192.168.65.10 | OpenPLC — Modbus TCP 502 |

## 3. Services d'infrastructure

| Service | Serveur | Portée |
|---|---|---|
| DNS interne (`aerosec.local`) | SRV-DC-01 | VLAN 10, 20, 30 |
| DNS forwarder | FW-AERO-01 | VLAN 50, 55 |
| DHCP | SRV-DC-01 (relais pfSense) | VLAN 20 uniquement |
| NTP | FW-AERO-01 → SRV-DC-01 → clients | Tous — **indispensable** pour la corrélation Wazuh |
| Résolution OT | fichiers `hosts` locaux | VLAN 60, 65 — aucun DNS traversant la frontière IT/OT |

> **Point d'attention** : sans NTP cohérent, les horodatages des logs seront décalés et
> la corrélation d'incidents dans Wazuh deviendra inexploitable. À configurer dès la Phase 3.

## 4. Configuration VirtualBox

### 4.1 Créer les réseaux internes

Aucune création préalable n'est nécessaire : un réseau interne VirtualBox existe dès qu'une
VM y est rattachée. Il suffit de saisir le nom exact (`aero-adm`, `aero-users`, …).

### 4.2 Interfaces de FW-AERO-01

L'interface graphique VirtualBox n'expose que 4 cartes réseau. Les cartes 5 à 8 se
configurent en ligne de commande, **VM éteinte** :

```bash
VBoxManage modifyvm "FW-AERO-01" --nic1 nat        --nictype1 82540EM
VBoxManage modifyvm "FW-AERO-01" --nic2 intnet --intnet2 "aero-adm"    --nictype2 82540EM
VBoxManage modifyvm "FW-AERO-01" --nic3 intnet --intnet3 "aero-users"  --nictype3 82540EM
VBoxManage modifyvm "FW-AERO-01" --nic4 intnet --intnet4 "aero-srv"    --nictype4 82540EM
VBoxManage modifyvm "FW-AERO-01" --nic5 intnet --intnet5 "aero-dmz"    --nictype5 82540EM
VBoxManage modifyvm "FW-AERO-01" --nic6 intnet --intnet6 "aero-idmz"   --nictype6 82540EM
VBoxManage modifyvm "FW-AERO-01" --nic7 intnet --intnet7 "aero-otsup"  --nictype7 82540EM
VBoxManage modifyvm "FW-AERO-01" --nic8 intnet --intnet8 "aero-otproc" --nictype8 82540EM
```

Vérification :

```bash
VBoxManage showvminfo "FW-AERO-01" | grep -i "NIC"
```

L'ordre des cartes VirtualBox correspond à l'ordre `em0` … `em7` sous pfSense. Noter les
adresses MAC avant l'installation pour éviter toute erreur d'affectation d'interface.

### 4.3 Interfaces des autres VM

Une seule carte, en mode `Réseau interne`, rattachée au segment de la machine.
**Aucune VM autre que le pare-feu ne doit disposer d'une carte NAT ou pont** — sinon la
segmentation est contournée et tout le lab perd son sens.

Contrôle à réaliser avant de passer à la phase suivante :

```bash
VBoxManage list vms | cut -d'"' -f2 | while read vm; do
  echo "=== $vm"; VBoxManage showvminfo "$vm" | grep -E "^NIC [1-8]:"
done
```

## 5. Étapes de mise en œuvre (Phase 3)

1. Installer pfSense sur FW-AERO-01, affecter `em0` en WAN.
2. Assigner et nommer les 7 interfaces LAN selon la table §1.
3. Configurer les IP statiques des passerelles.
4. Désactiver la règle « allow all » par défaut sur chaque interface LAN.
5. Créer les alias pfSense (voir `Flow_Matrix.md` §2).
6. Appliquer les règles de la matrice des flux.
7. Tester la connectivité intra-zone, puis l'isolation inter-zones.

## 6. Preuves à collecter (dossier `08_Evidence/`)

- Capture de la page *Interfaces → Assignments* de pfSense
- Sortie de `VBoxManage showvminfo` pour chaque VM
- `ping` réussi intra-zone / échec inter-zone (VLAN 20 → VLAN 60 notamment)
- Table de routage pfSense (*Diagnostics → Routes*)
