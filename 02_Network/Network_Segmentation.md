# Matrice des flux et politique pare-feu — AEROSEC INDUSTRIES

**Principe fondateur : `deny by default`.** Chaque interface du pare-feu se termine par une
règle de blocage explicite avec journalisation. Tout flux autorisé doit avoir un
propriétaire, une justification métier et un identifiant de règle.

## 1. Matrice de synthèse

Lecture : ligne = source, colonne = destination.
`✔` autorisé (ports restreints) · `✖` interdit · `◐` autorisé sous conditions

| Source \ Destination | WAN | DMZ (50) | USERS (20) | SERVERS (30) | ADM-SOC (10) | IDMZ (55) | OT-SUP (60) | OT-PROC (65) |
|---|---|---|---|---|---|---|---|---|
| **WAN** | — | ✔ | ✖ | ✖ | ✖ | ✖ | ✖ | ✖ |
| **DMZ (50)** | ◐ | — | ✖ | ✖ | ✔ (logs) | ✖ | ✖ | ✖ |
| **USERS (20)** | ✔ | ✔ | — | ✔ | ✔ (logs) | ◐ | ✖ | ✖ |
| **SERVERS (30)** | ◐ | ✖ | ✖ | — | ✔ (logs) | ✖ | ✖ | ✖ |
| **ADM-SOC (10)** | ◐ | ✔ | ✔ | ✔ | — | ✔ | ✔ | ◐ |
| **IDMZ (55)** | ✖ | ✖ | ✖ | ✖ | ✔ (logs) | — | ✔ | ✖ |
| **OT-SUP (60)** | ✖ | ✖ | ✖ | ✖ | ✔ (logs) | ✔ | — | ✔ |
| **OT-PROC (65)** | ✖ | ✖ | ✖ | ✖ | ✔ (logs) | ✖ | ✔ (réponses) | — |

**Les deux règles à retenir et à savoir défendre :**

1. Aucune case ✔ entre `USERS`/`SERVERS` et `OT-SUP`/`OT-PROC`. La seule voie est
   `USERS → IDMZ → OT-SUP` (conduit IEC 62443), et uniquement via le bastion.
2. `OT-PROC` ne parle qu'à `OT-SUP` et au SIEM. Jamais à Internet, jamais à l'IT.

## 2. Alias pfSense à créer

| Alias | Type | Contenu |
|---|---|---|
| `NET_USERS` | Réseau | 192.168.20.0/24 |
| `NET_SERVERS` | Réseau | 192.168.30.0/24 |
| `NET_OT` | Réseau | 192.168.60.0/24, 192.168.65.0/24 |
| `HOST_DC` | Hôte | 192.168.30.10 |
| `HOST_SIEM` | Hôte | 192.168.10.10 |
| `HOST_JUMP` | Hôte | 192.168.55.10 |
| `HOST_PLC` | Hôte | 192.168.65.10 |
| `PORTS_AD` | Ports | 53, 88, 123, 135, 389, 445, 464, 636, 3268, 49152:65535 |
| `PORTS_WAZUH` | Ports | 514, 1514, 1515 |
| `PORTS_ADMIN` | Ports | 22, 3389, 5985, 5986 |
| `PORTS_ICS` | Ports | 502, 8080 |

## 3. Règles détaillées par interface

### 3.1 WAN (`em0`)

| ID | Action | Source | Destination | Port | Log | Justification |
|---|---|---|---|---|---|---|
| WAN-01 | Bloquer | Réseaux privés / bogons | any | any | ✔ | Anti-spoofing |
| WAN-02 | Autoriser | any | DMZ-WEB-01 | 80, 443 | ✔ | Portail public (NAT entrant) |
| WAN-99 | Bloquer | any | any | any | ✔ | Deny by default |

### 3.2 DMZ (`em4`, VLAN 50)

| ID | Action | Source | Destination | Port | Log | Justification |
|---|---|---|---|---|---|---|
| DMZ-01 | Autoriser | NET_DMZ | HOST_SIEM | PORTS_WAZUH | — | Remontée des logs |
| DMZ-02 | Autoriser | NET_DMZ | any (WAN) | 80, 443 | ✔ | Mises à jour système |
| DMZ-99 | Bloquer | NET_DMZ | any | any | ✔ | Une DMZ compromise ne doit jamais atteindre l'IT |

### 3.3 USERS (`em2`, VLAN 20)

| ID | Action | Source | Destination | Port | Log | Justification |
|---|---|---|---|---|---|---|
| USR-01 | Autoriser | NET_USERS | HOST_DC | PORTS_AD | — | Authentification, DNS, GPO |
| USR-02 | Autoriser | NET_USERS | SRV-FS-01 | 445 | — | Partages de fichiers |
| USR-03 | Autoriser | NET_USERS | HOST_SIEM | PORTS_WAZUH | — | Agents Wazuh |
| USR-04 | Autoriser | NET_USERS | any (WAN) | 80, 443, 53 | ✔ | Navigation |
| USR-05 | Autoriser | `GG_OT_Access` (IP du poste) | HOST_JUMP | 443 | ✔ | Accès bastion, groupe restreint |
| USR-90 | **Bloquer** | NET_USERS | NET_OT | any | ✔ | **Scénario 5 — tentative IT → OT** |
| USR-99 | Bloquer | NET_USERS | any | any | ✔ | Deny by default |

> `USR-90` est placée **avant** la règle finale et journalisée explicitement : c'est cette
> règle qui alimentera l'alerte Wazuh du scénario 5. Ne pas se contenter du deny implicite.

### 3.4 SERVERS (`em3`, VLAN 30)

| ID | Action | Source | Destination | Port | Log | Justification |
|---|---|---|---|---|---|---|
| SRV-01 | Autoriser | NET_SERVERS | HOST_SIEM | PORTS_WAZUH | — | Remontée des logs |
| SRV-02 | Autoriser | HOST_DC | any (WAN) | 53, 123, 443 | ✔ | DNS récursif, NTP, mises à jour |
| SRV-03 | Autoriser | NET_SERVERS | NET_USERS | établies uniquement | — | Réponses aux clients |
| SRV-99 | Bloquer | NET_SERVERS | any | any | ✔ | Deny by default |

### 3.5 ADM-SOC (`em1`, VLAN 10)

| ID | Action | Source | Destination | Port | Log | Justification |
|---|---|---|---|---|---|---|
| ADM-01 | Autoriser | WKS-ADM-01 | any | PORTS_ADMIN, 443 | ✔ | Administration depuis le PAW uniquement |
| ADM-02 | Autoriser | HOST_SIEM | any | 1514, 1515, 55000 | — | Déploiement et pilotage des agents |
| ADM-03 | Autoriser | SRV-BKP-01 | NET_SERVERS | 22, 445 | ✔ | Sauvegarde en mode *pull* |
| ADM-04 | Autoriser | NET_ADM | any (WAN) | 80, 443 | ✔ | Mises à jour |
| ADM-90 | Bloquer | NET_ADM | HOST_PLC | 502 | ✔ | Écriture Modbus interdite depuis l'IT, même en admin |
| ADM-99 | Bloquer | NET_ADM | any | any | ✔ | Deny by default |

### 3.6 IDMZ (`em5`, VLAN 55) — le conduit

| ID | Action | Source | Destination | Port | Log | Justification |
|---|---|---|---|---|---|---|
| IDZ-01 | Autoriser | HOST_JUMP | OT-SCADA-01, WKS-ENG-01 | 22, 3389 | ✔ | Rebond vers l'OT |
| IDZ-02 | Autoriser | HOST_JUMP | HOST_SIEM | PORTS_WAZUH | — | Journalisation des sessions |
| IDZ-90 | Bloquer | NET_IDMZ | NET_SERVERS, NET_USERS | any | ✔ | Pas de retour vers l'IT |
| IDZ-91 | Bloquer | NET_IDMZ | any (WAN) | any | ✔ | Pas de sortie Internet |
| IDZ-99 | Bloquer | NET_IDMZ | any | any | ✔ | Deny by default |

### 3.7 OT-SUP (`em6`, VLAN 60)

| ID | Action | Source | Destination | Port | Log | Justification |
|---|---|---|---|---|---|---|
| OTS-01 | Autoriser | OT-SCADA-01 | HOST_PLC | 502, 8080 | ✔ | Modbus TCP, interface OpenPLC |
| OTS-02 | Autoriser | NET_OTSUP | HOST_JUMP | 443 | ✔ | Push historian vers l'IDMZ |
| OTS-03 | Autoriser | NET_OTSUP | HOST_SIEM | PORTS_WAZUH | — | Agents Wazuh |
| OTS-90 | Bloquer | NET_OTSUP | any (WAN) | any | ✔ | **Aucun accès Internet depuis l'OT** |
| OTS-99 | Bloquer | NET_OTSUP | any | any | ✔ | Deny by default |

### 3.8 OT-PROC (`em7`, VLAN 65)

| ID | Action | Source | Destination | Port | Log | Justification |
|---|---|---|---|---|---|---|
| OTP-01 | Autoriser | HOST_PLC | HOST_SIEM | PORTS_WAZUH | — | Journalisation de l'automate |
| OTP-99 | Bloquer | HOST_PLC | any | any | ✔ | L'automate n'initie aucune connexion sortante |

## 4. Suricata

| Interface | Mode | Jeux de règles |
|---|---|---|
| `em0` (WAN) | IDS puis IPS | ET Open — scan, exploit, malware C2 |
| `em6` (OT-SUP) | IDS uniquement | ET Open SCADA + règles Modbus personnalisées |
| `em2` (USERS) | IDS | ET Open — mouvement latéral, reconnaissance |

Mode IDS seul sur l'OT : un faux positif en IPS couperait la supervision du procédé —
en environnement industriel, la disponibilité prime sur la confidentialité (inversion
de la triade DIC, argument classique en entretien aéronautique).

## 5. Plan de tests de validation

| Test | Commande | Résultat attendu |
|---|---|---|
| T1 | `WKS-IT-01` → `ping 192.168.60.10` | Échec + entrée de log règle USR-90 |
| T2 | `WKS-IT-01` → `nslookup srv-dc-01.aerosec.local` | Succès |
| T3 | `OT-SCADA-01` → `curl https://google.com` | Échec (règle OTS-90) |
| T4 | `OT-SCADA-01` → `nc -zv 192.168.65.10 502` | Succès |
| T5 | `WKS-IT-01` → RDP sur `SRV-DC-01` | Échec (admin depuis PAW uniquement) |
| T6 | `WKS-ADM-01` → RDP sur `SRV-DC-01` | Succès |
| T7 | `DMZ-WEB-01` → `ping 192.168.30.10` | Échec (règle DMZ-99) |
| T8 | Toutes les VM → agent Wazuh | Statut *Active* dans le tableau de bord |

Chaque test doit être capturé (écran ou sortie console) et déposé dans
`08_Evidence/` avec l'identifiant du test dans le nom du fichier.

## 6. Correspondance avec les référentiels

| Règle / principe | Référentiel |
|---|---|
| Deny by default sur toutes les interfaces | ISO 27002 §8.20, NIST CSF PR.AC-5 |
| Segmentation en zones et conduits | IEC 62443-3-2 |
| Aucun flux IT → OT direct | IEC 62443-3-3 SR 5.1 / SR 5.2 |
| Administration depuis un poste dédié | ISO 27002 §8.2, modèle *tiering* Microsoft |
| Journalisation des refus | ISO 27002 §8.15, NIST CSF DE.AE-3 |
| Absence d'accès Internet depuis l'OT | IEC 62443-3-3 SR 5.4 |
