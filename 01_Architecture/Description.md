# Description de l'architecture â€” AEROSEC INDUSTRIES

## 1. SchÃ©ma logique

```
                            INTERNET
                                â”‚
                          â”Œâ”€â”€â”€â”€â”€â”´â”€â”€â”€â”€â”€â”
                          â”‚ AER-FW01  â”‚  pfSense + Suricata
                          â”‚  7 zones  â”‚  deny by default
                          â””â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”˜
        â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”¼â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
        â”‚          â”‚            â”‚            â”‚          â”‚
      DMZ       SERVERS      USERS          SOC        OT
   192.168.50    192.168.30    192.168.20     192.168.40        â”‚
        â”‚          â”‚            â”‚            â”‚          â”‚
   (portail)   AER-DC01     AER-WS01    AER-SOC01       â”‚
                AD/DNS      Win 11       Wazuh          â”‚
                DHCP        IngÃ©nierie   Suricata       â”‚
                Partages    AER-KALI01   Dashboard      â”‚
                                                        â”‚
                                        â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”´â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
                                        â”‚                              â”‚
                                     OT â€” Niveau 2               OT â€” Niveau 1
                                      192.168.60                     192.168.65
                                        â”‚                              â”‚
                                  AER-SCADA01  â”€â”€â”€â”€ Modbus/TCP â”€â”€â–¶ AER-PLC01
                                   FUXA (IHM)         port 502        OpenPLC
                                   Supervision                        Automate
                                                                          â”‚
                                                                    ProcÃ©dÃ© simulÃ©
                                                          (convoyage Â· usinage Â· contrÃ´le)
```

## 2. Justification des choix de conception

### 2.1 Sept zones plutÃ´t qu'une sÃ©paration IT/OT binaire

Un dÃ©coupage en deux zones aurait suffi Ã  rÃ©pondre littÃ©ralement Ã  l'Ã©noncÃ©, mais il aurait produit une architecture peu crÃ©dible. Le dÃ©coupage retenu permet de :

- isoler les **serveurs** des **postes utilisateurs**, de sorte que la compromission d'un poste n'ouvre pas un accÃ¨s latÃ©ral libre vers l'annuaire ;
- placer le **SOC** hors du pÃ©rimÃ¨tre de production, afin que les journaux restent exploitables mÃªme si le SI bureautique est compromis ;
- distinguer les **deux niveaux OT**, ce qui restreint l'Ã©criture Modbus Ã  la seule station de supervision ;
- exposer les services publics en **DMZ** sans jamais ouvrir de flux entrant vers le rÃ©seau interne.

### 2.2 Suricata en package pfSense plutÃ´t qu'en machine dÃ©diÃ©e

Trois raisons :

1. **Positionnement** : l'IDS voit l'intÃ©gralitÃ© du trafic inter-zones, y compris les flux bloquÃ©s, ce qu'une sonde placÃ©e dans une seule zone ne permettrait pas.
2. **Ressources** : l'hÃ´te dispose de 16 Go de mÃ©moire ; Ã©conomiser une machine virtuelle complÃ¨te est dÃ©terminant.
3. **RÃ©alisme** : c'est le dÃ©ploiement le plus courant en PME industrielle, oÃ¹ l'IDS est frÃ©quemment adossÃ© au pare-feu pÃ©rimÃ©trique.

### 2.3 SÃ©paration OT en deux niveaux

Sans cette sÃ©paration, la rÃ¨gle Â« seule la supervision peut Ã©crire vers l'automate Â» serait inapplicable, puisque toutes les machines industrielles partageraient le mÃªme segment. Avec elle, la tentative d'Ã©criture Modbus depuis un poste bureautique doit traverser deux frontiÃ¨res de filtrage â€” elle est donc bloquÃ©e deux fois et journalisÃ©e deux fois, ce qui rend le scÃ©nario 5 dÃ©montrable.

C'est la traduction directe du concept de **zones et conduits** de l'IEC 62443.

### 2.4 Absence de DMZ industrielle

Une architecture de production intercalerait une DMZ industrielle entre les niveaux 3 et 4, hÃ©bergeant un historian et un serveur de saut. Elle n'est pas implÃ©mentÃ©e ici faute de ressources mÃ©moire.

Cet Ã©cart est assumÃ© et documentÃ© : le flux F-12 (consultation de l'IHM depuis le poste bureautique) constitue prÃ©cisÃ©ment le flux qui, en production, transiterait par cette DMZ industrielle via un serveur de rebond avec authentification renforcÃ©e.

### 2.5 Adressage statique en zone OT

Aucun serveur DHCP n'est dÃ©ployÃ© dans les zones industrielles. Chaque Ã©quipement est dÃ©clarÃ© explicitement, ce qui rend immÃ©diatement anormale l'apparition d'une machine non rÃ©pertoriÃ©e â€” signal exploitÃ© en dÃ©tection.

## 3. Ordre de construction

| Phase | Contenu | DÃ©pendance |
|---|---|---|
| 2 | CrÃ©ation et adaptation des machines virtuelles | Phase 1 |
| 3 | pfSense, interfaces, rÃ¨gles de filtrage | Phase 2 |
| 4 | Active Directory, unitÃ©s d'organisation, comptes | Phase 3 |
| 5 | Partages de fichiers et sauvegardes | Phase 4 |
| 6 | Automate, supervision, liaison Modbus | Phase 3 |
| 7 | Wazuh, agents, Sysmon | Phases 4 et 6 |
| 8 | Suricata et rÃ¨gles rÃ©seau | Phase 7 |
| 9 | ScÃ©narios d'attaque â€” Ã©tat initial | Phase 8 |
| 10 | Durcissement GPO, AD, OT, rÃ©seau | Phase 9 |
| 11 | RÃ¨gles de dÃ©tection et rejeu des scÃ©narios | Phase 10 |
| 12 | Analyse de risques et plan de remÃ©diation | Phase 11 |
| 13 | Rapport et prÃ©sentation | Phase 12 |

Le SIEM est dÃ©ployÃ© **avant** le durcissement, afin de disposer d'un Ã©tat de rÃ©fÃ©rence : les scÃ©narios de la Phase 9 dÃ©montrent ce qui passe inaperÃ§u sur un systÃ¨me non durci, ceux de la Phase 11 dÃ©montrent la dÃ©tection aprÃ¨s correction. Cette comparaison avant/aprÃ¨s constitue la dÃ©monstration la plus convaincante du dossier.

## 4. Mesures de sÃ©curitÃ© prÃ©vues par zone

| Zone | Mesures |
|---|---|
| PÃ©rimÃ¨tre | Filtrage deny by default, journalisation des refus, dÃ©tection Suricata |
| SERVERS | Durcissement AD, comptes Ã  privilÃ¨ges sÃ©parÃ©s, audit des accÃ¨s fichiers, sauvegarde testÃ©e |
| USERS | GPO de durcissement, Defender, pare-feu local, restriction USB, Sysmon, journalisation PowerShell |
| SOC | Collecte centralisÃ©e, rÃ¨gles de dÃ©tection personnalisÃ©es, mapping MITRE ATT&CK, playbooks de rÃ©ponse |
| OT-L2 | Restriction des flux entrants, horodatage fiable, supervision des alarmes |
| OT-L1 | Liste blanche d'Ã©criture Modbus, aucune sortie Internet, journalisation de toute tentative d'accÃ¨s |

## 5. RÃ©fÃ©rentiels mobilisÃ©s

| RÃ©fÃ©rentiel | Usage dans le projet |
|---|---|
| **ISO/IEC 27001:2022** | Mesures de l'Annexe A, analyse de risques, gouvernance |
| **ISO/IEC 27002:2022** | Guide de mise en Å“uvre des mesures |
| **NIST CSF** | Structuration en fonctions Identifier / ProtÃ©ger / DÃ©tecter / RÃ©pondre / RÃ©cupÃ©rer |
| **IEC 62443** | Zones et conduits, exigences de sÃ©curitÃ© systÃ¨me en environnement industriel |
| **ModÃ¨le de Purdue** | Structuration hiÃ©rarchique de l'architecture OT |
| **MITRE ATT&CK** | CaractÃ©risation des techniques d'attaque IT |
| **MITRE ATT&CK for ICS** | CaractÃ©risation des techniques d'attaque industrielles |

Chaque mesure appliquÃ©e est rattachÃ©e Ã  au moins un contrÃ´le d'un de ces rÃ©fÃ©rentiels dans le rapport final. Citer un rÃ©fÃ©rentiel sans montrer le contrÃ´le correspondant n'apporte rien.

---

*Document rÃ©digÃ© en Phase 1 â€” Architecture. Le schÃ©ma dÃ©taillÃ© sous draw.io accompagne ce document (`Architecture.drawio` et `Architecture.png`).*

