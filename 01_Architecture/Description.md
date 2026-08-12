# Description de l'architecture — AEROSEC INDUSTRIES

## 1. Schéma logique

```
                            INTERNET
                                │
                          ┌─────┴─────┐
                          │ AER-FW01  │  pfSense + Suricata
                          │  7 zones  │  deny by default
                          └─────┬─────┘
        ┌──────────┬────────────┼────────────┬──────────┐
        │          │            │            │          │
      DMZ       SERVERS      USERS          SOC        OT
   10.10.50    10.10.30    10.10.20     10.10.40        │
        │          │            │            │          │
   (portail)   AER-DC01     AER-WS01    AER-SOC01       │
                AD/DNS      Win 11       Wazuh          │
                DHCP        Ingénierie   Suricata       │
                Partages    AER-KALI01   Dashboard      │
                                                        │
                                        ┌───────────────┴──────────────┐
                                        │                              │
                                     OT — Niveau 2               OT — Niveau 1
                                      10.10.60                     10.10.61
                                        │                              │
                                  AER-SCADA01  ──── Modbus/TCP ──▶ AER-PLC01
                                   FUXA (IHM)         port 502        OpenPLC
                                   Supervision                        Automate
                                                                          │
                                                                    Procédé simulé
                                                          (convoyage · usinage · contrôle)
```

## 2. Justification des choix de conception

### 2.1 Sept zones plutôt qu'une séparation IT/OT binaire

Un découpage en deux zones aurait suffi à répondre littéralement à l'énoncé, mais il aurait produit une architecture peu crédible. Le découpage retenu permet de :

- isoler les **serveurs** des **postes utilisateurs**, de sorte que la compromission d'un poste n'ouvre pas un accès latéral libre vers l'annuaire ;
- placer le **SOC** hors du périmètre de production, afin que les journaux restent exploitables même si le SI bureautique est compromis ;
- distinguer les **deux niveaux OT**, ce qui restreint l'écriture Modbus à la seule station de supervision ;
- exposer les services publics en **DMZ** sans jamais ouvrir de flux entrant vers le réseau interne.

### 2.2 Suricata en package pfSense plutôt qu'en machine dédiée

Trois raisons :

1. **Positionnement** : l'IDS voit l'intégralité du trafic inter-zones, y compris les flux bloqués, ce qu'une sonde placée dans une seule zone ne permettrait pas.
2. **Ressources** : l'hôte dispose de 16 Go de mémoire ; économiser une machine virtuelle complète est déterminant.
3. **Réalisme** : c'est le déploiement le plus courant en PME industrielle, où l'IDS est fréquemment adossé au pare-feu périmétrique.

### 2.3 Séparation OT en deux niveaux

Sans cette séparation, la règle « seule la supervision peut écrire vers l'automate » serait inapplicable, puisque toutes les machines industrielles partageraient le même segment. Avec elle, la tentative d'écriture Modbus depuis un poste bureautique doit traverser deux frontières de filtrage — elle est donc bloquée deux fois et journalisée deux fois, ce qui rend le scénario 5 démontrable.

C'est la traduction directe du concept de **zones et conduits** de l'IEC 62443.

### 2.4 Absence de DMZ industrielle

Une architecture de production intercalerait une DMZ industrielle entre les niveaux 3 et 4, hébergeant un historian et un serveur de saut. Elle n'est pas implémentée ici faute de ressources mémoire.

Cet écart est assumé et documenté : le flux F-12 (consultation de l'IHM depuis le poste bureautique) constitue précisément le flux qui, en production, transiterait par cette DMZ industrielle via un serveur de rebond avec authentification renforcée.

### 2.5 Adressage statique en zone OT

Aucun serveur DHCP n'est déployé dans les zones industrielles. Chaque équipement est déclaré explicitement, ce qui rend immédiatement anormale l'apparition d'une machine non répertoriée — signal exploité en détection.

## 3. Ordre de construction

| Phase | Contenu | Dépendance |
|---|---|---|
| 2 | Création et adaptation des machines virtuelles | Phase 1 |
| 3 | pfSense, interfaces, règles de filtrage | Phase 2 |
| 4 | Active Directory, unités d'organisation, comptes | Phase 3 |
| 5 | Partages de fichiers et sauvegardes | Phase 4 |
| 6 | Automate, supervision, liaison Modbus | Phase 3 |
| 7 | Wazuh, agents, Sysmon | Phases 4 et 6 |
| 8 | Suricata et règles réseau | Phase 7 |
| 9 | Scénarios d'attaque — état initial | Phase 8 |
| 10 | Durcissement GPO, AD, OT, réseau | Phase 9 |
| 11 | Règles de détection et rejeu des scénarios | Phase 10 |
| 12 | Analyse de risques et plan de remédiation | Phase 11 |
| 13 | Rapport et présentation | Phase 12 |

Le SIEM est déployé **avant** le durcissement, afin de disposer d'un état de référence : les scénarios de la Phase 9 démontrent ce qui passe inaperçu sur un système non durci, ceux de la Phase 11 démontrent la détection après correction. Cette comparaison avant/après constitue la démonstration la plus convaincante du dossier.

## 4. Mesures de sécurité prévues par zone

| Zone | Mesures |
|---|---|
| Périmètre | Filtrage deny by default, journalisation des refus, détection Suricata |
| SERVERS | Durcissement AD, comptes à privilèges séparés, audit des accès fichiers, sauvegarde testée |
| USERS | GPO de durcissement, Defender, pare-feu local, restriction USB, Sysmon, journalisation PowerShell |
| SOC | Collecte centralisée, règles de détection personnalisées, mapping MITRE ATT&CK, playbooks de réponse |
| OT-L2 | Restriction des flux entrants, horodatage fiable, supervision des alarmes |
| OT-L1 | Liste blanche d'écriture Modbus, aucune sortie Internet, journalisation de toute tentative d'accès |

## 5. Référentiels mobilisés

| Référentiel | Usage dans le projet |
|---|---|
| **ISO/IEC 27001:2022** | Mesures de l'Annexe A, analyse de risques, gouvernance |
| **ISO/IEC 27002:2022** | Guide de mise en œuvre des mesures |
| **NIST CSF** | Structuration en fonctions Identifier / Protéger / Détecter / Répondre / Récupérer |
| **IEC 62443** | Zones et conduits, exigences de sécurité système en environnement industriel |
| **Modèle de Purdue** | Structuration hiérarchique de l'architecture OT |
| **MITRE ATT&CK** | Caractérisation des techniques d'attaque IT |
| **MITRE ATT&CK for ICS** | Caractérisation des techniques d'attaque industrielles |

Chaque mesure appliquée est rattachée à au moins un contrôle d'un de ces référentiels dans le rapport final. Citer un référentiel sans montrer le contrôle correspondant n'apporte rien.

---

*Document rédigé en Phase 1 — Architecture. Le schéma détaillé sous draw.io accompagne ce document (`Architecture.drawio` et `Architecture.png`).*
