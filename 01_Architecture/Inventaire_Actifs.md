# Inventaire des actifs — AEROSEC INDUSTRIES

## 1. Méthode de cotation

Chaque actif est coté sur trois critères, de 1 (faible) à 4 (critique) :

- **D — Disponibilité** : impact d'une indisponibilité sur l'activité ;
- **I — Intégrité** : impact d'une altération non détectée des données ;
- **C — Confidentialité** : impact d'une divulgation non autorisée.

La **criticité globale** retenue est la valeur maximale des trois critères, conformément à l'approche par le pire cas utilisée en analyse de risques.

| Niveau | Libellé | Interprétation |
|---|---|---|
| 1 | Mineur | Gêne opérationnelle, sans impact métier |
| 2 | Modéré | Impact interne, contournement possible |
| 3 | Majeur | Impact sur les engagements clients |
| 4 | Critique | Arrêt d'activité, non-conformité réglementaire ou risque produit |

## 2. Actifs primaires (informations et processus)

| ID | Actif | Type | D | I | C | Criticité | Propriétaire |
|---|---|---|---|---|---|---|---|
| AP-01 | Plans et modèles 3D | Information | 3 | 4 | 4 | **4** | Direction technique |
| AP-02 | Gammes d'usinage et programmes machine | Information | 4 | 4 | 3 | **4** | Production |
| AP-03 | Dossiers de conformité pièces | Information | 3 | 4 | 2 | **4** | Qualité |
| AP-04 | Spécifications clients | Information | 2 | 3 | 4 | **4** | Direction technique |
| AP-05 | Données RH et paie | Information | 2 | 3 | 4 | **4** | DRH |
| AP-06 | Données comptables | Information | 2 | 4 | 3 | **4** | DAF |
| AP-07 | Processus de production continue | Processus | 4 | 4 | 1 | **4** | Production |
| AP-08 | Annuaire des identités | Information | 4 | 4 | 3 | **4** | DSI |
| AP-09 | Journaux de sécurité | Information | 2 | 4 | 3 | **4** | RSSI |

## 3. Actifs supports (systèmes du laboratoire)

| ID | Nom | Zone | Rôle | OS | D | I | C | Criticité |
|---|---|---|---|---|---|---|---|---|
| AS-01 | AER-FW01 | Périmètre | Pare-feu, routage inter-zones, IDS | pfSense CE 2.7 | 4 | 4 | 2 | **4** |
| AS-02 | AER-DC01 | SERVERS | Contrôleur de domaine, DNS, DHCP, partages | Windows Server 2022 | 4 | 4 | 3 | **4** |
| AS-03 | AER-WS01 | USERS | Poste utilisateur / station d'ingénierie | Windows 11 Enterprise | 2 | 3 | 3 | **3** |
| AS-04 | AER-SOC01 | SOC | SIEM Wazuh (indexer, manager, dashboard) | Ubuntu Server | 3 | 4 | 3 | **4** |
| AS-05 | AER-SCADA01 | OT niveau 2 | Supervision et IHM (FUXA) | Ubuntu Server | 4 | 4 | 2 | **4** |
| AS-06 | AER-PLC01 | OT niveau 1 | Automate simulé (OpenPLC), Modbus/TCP | Ubuntu Server | 4 | 4 | 1 | **4** |
| AS-07 | AER-KALI01 | USERS (test) | Machine de test contrôlée | Kali Linux | 1 | 1 | 1 | **1** |

## 4. Correspondance avec les VM existantes

| Actif | VM d'origine | Action en Phase 2 |
|---|---|---|
| AER-FW01 | — | Création depuis l'ISO pfSense |
| AER-DC01 | `windows2022DC01` | Réutilisation, renommage, reconfiguration sur `aerosec.local` |
| AER-WS01 | — | Création depuis l'ISO Windows 11 Enterprise |
| AER-SOC01 | `Ubuntu` | Clonage puis installation de Wazuh |
| AER-SCADA01 | `OT-FUXA` | Réutilisation et renommage |
| AER-PLC01 | `OT-OpenPLC 1` | Réutilisation et renommage |
| AER-KALI01 | `kali-linux-2026.1-virtualbox-amd64` | Réutilisation et renommage |

## 5. Positionnement dans le modèle de Purdue

| Niveau | Désignation | Actifs AEROSEC |
|---|---|---|
| **Niveau 4/5** | Systèmes d'entreprise (IT) | AER-DC01, AER-WS01 |
| **DMZ industrielle** | Zone tampon IT/OT | *(non implémentée — voir limites)* |
| **Niveau 3** | Gestion des opérations | *(fonction assurée par AER-SCADA01)* |
| **Niveau 2** | Supervision et contrôle | AER-SCADA01 (IHM, supervision) |
| **Niveau 1** | Contrôle local | AER-PLC01 (automate) |
| **Niveau 0** | Procédé physique | Simulé logiciellement dans OpenPLC |

La zone SOC est transversale : elle collecte les journaux de tous les niveaux sans appartenir à la chaîne de production.

## 6. Actifs les plus exposés

Ces trois actifs concentrent l'essentiel du risque et orientent les priorités de durcissement :

1. **AER-DC01** — compromettre le contrôleur de domaine revient à compromettre l'ensemble du SI bureautique. Sa colocalisation avec le serveur de fichiers aggrave l'exposition.
2. **AER-PLC01** — le protocole Modbus/TCP ne comporte ni authentification ni chiffrement. Toute machine capable de l'atteindre sur le port 502 peut écrire dans ses registres.
3. **AER-WS01** — point d'entrée le plus probable d'une intrusion (hameçonnage, support amovible), et disposant d'un accès légitime aux données de conception.

---

*Document rédigé en Phase 1 — Architecture. Ce tableau constitue la base de la matrice des risques de la Phase 12.*
