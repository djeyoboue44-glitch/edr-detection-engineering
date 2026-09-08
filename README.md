# 🛡️ edr-detection-engineering

> MITRE ATT&CK detection lab — Wazuh SIEM/EDR, Sysmon telemetry, adversary emulation with Atomic Red Team, and custom detection rule engineering.

![Status](https://img.shields.io/badge/status-completed-brightgreen)
![Wazuh](https://img.shields.io/badge/Wazuh-4.13.1-blue)
![MITRE ATT%26CK](https://img.shields.io/badge/MITRE-ATT%26CK-red)

---

## 🎯 Objectif du projet

Reproduire, dans un environnement isolé, le cycle complet d'un analyste SOC / détection engineer :
1. Déployer un SIEM/EDR (Wazuh) capable d'ingérer et d'analyser des logs Windows en temps réel.
2. Instrumenter un poste cible avec une télémétrie de qualité (Sysmon).
3. Simuler des techniques d'attaque réelles et documentées (Atomic Red Team, mappées MITRE ATT&CK).
4. Vérifier ce que le SIEM détecte nativement, identifier les angles morts, et **écrire une règle de détection personnalisée** pour combler un manque.
5. Valider la règle dans la durée, pas seulement au moment du test.

Projet réalisé seul, du 07/09/2026 16h au 08/09/2026 21h, dans le cadre de ma préparation à une alternance en cybersécurité (SOC / Blue Team).

---

## 🏗️ Architecture du lab

\`\`\`
                     ┌─────────────────────────────┐
                     │   VM Ubuntu — "projet 2"      │
                     │   Wazuh All-in-One             │
                     │   (Indexer + Manager +          │
                     │    Dashboard) — v4.13.1          │
                     │   192.168.237.154                │
                     └───────────────▲─────────────────┘
                                     │ Agent Wazuh
                                     │ (port 1514/1515)
                     ┌───────────────┴─────────────────┐
                     │   VM Windows 10 Pro — WIN10-TARGET│
                     │   Sysmon (config SwiftOnSecurity) │
                     │   Atomic Red Team                 │
                     │   192.168.237.152                 │
                     └───────────────────────────────────┘
\`\`\`

- **Hyperviseur** : VMware Workstation (bascule depuis VirtualBox en cours de route)
- **VM SIEM** : Ubuntu 26.04.1 LTS — 8 vCPU / ~7,2 Go RAM / 49 Go disque
- **VM cible** : Windows 10 Professionnel — 2 vCPU / 4 Go RAM
- Réseau NAT, isolé de l'hôte

---

## 🔧 Stack technique

| Composant | Rôle |
|---|---|
| **Wazuh 4.13.1** (Indexer, Manager, Dashboard) | SIEM/EDR — collecte, corrélation, alerting |
| **Sysmon v15.21** (config [SwiftOnSecurity](https://github.com/SwiftOnSecurity/sysmon-config)) | Télémétrie Windows haute-fidélité (process creation, réseau, fichiers) |
| **Atomic Red Team** (Invoke-AtomicRedTeam) | Simulation d'attaques mappées MITRE ATT&CK |
| **PowerShell / Bash** | Automatisation et exploitation des logs |

---

## 🧪 Déroulé du projet

### 1. Déploiement du SIEM

Installation de Wazuh en mode *all-in-one* plutôt qu'une stack ELK montée à la main, pour bénéficier des règles de détection MITRE ATT&CK déjà intégrées. Premier disque de 20 Go insuffisant → redimensionné à 49 Go.

<p align="center">
  <img src="assets/images/01-wazuh-install-log-1.png" width="45%" />
  <img src="assets/images/02-wazuh-install-log-2.png" width="45%" />
</p>
<p align="center"><img src="assets/images/03-wazuh-login.png" width="60%" /></p>

### 2. Déploiement de l'agent

Agent Wazuh installé sur la VM Windows (`WIN10-TARGET`), enregistré et actif dans le Dashboard.

<p align="center">
  <img src="assets/images/04-wazuh-dashboard-no-agents.png" width="45%" />
  <img src="assets/images/05-deploy-agent-windows.png" width="45%" />
</p>
<p align="center">
  <img src="assets/images/06-agent-install-powershell.png" width="45%" />
  <img src="assets/images/07-wazuh-dashboard-agent-active.png" width="45%" />
</p>
<p align="center"><img src="assets/images/08-endpoints-agent-active.png" width="80%" /></p>

### 3. Instrumentation avec Sysmon

Installation de Sysmon avec la configuration communautaire SwiftOnSecurity (référence du secteur), pour obtenir une télémétrie exploitable (process creation avec ligne de commande complète, notamment).

<p align="center">
  <img src="assets/images/09-sysmon-download-page.png" width="45%" />
  <img src="assets/images/10-sysmonconfig-swiftonsecurity.png" width="45%" />
</p>
<p align="center">
  <img src="assets/images/11-sysmon-install-output.png" width="45%" />
  <img src="assets/images/12-sysmon-events-check.png" width="45%" />
</p>
<p align="center"><img src="assets/images/13-wazuh-discover-sysmon-events.png" width="80%" /></p>

### 4. Simulation d'attaque — Atomic Red Team

Installation d'Invoke-AtomicRedTeam et exécution de plusieurs sous-techniques de **T1082 — System Information Discovery** (`systeminfo`, requêtes registre, WMIC, découverte de comptes...).

<p align="center">
  <img src="assets/images/14-atomicredteam-install-start.png" width="45%" />
  <img src="assets/images/15-atomicredteam-install-complete.png" width="45%" />
</p>
<p align="center"><img src="assets/images/16-invoke-atomictest-t1082-list.png" width="80%" /></p>
<p align="center">
  <img src="assets/images/17-invoke-atomictest-t1082-1-output.png" width="45%" />
  <img src="assets/images/18-invoke-atomictest-t1082-1-done.png" width="45%" />
</p>
<p align="center"><img src="assets/images/21-atomictest-t1082-series-run.png" width="80%" /></p>

> Certains sous-tests (Connect-AzAccount, Connect-AzureAD, Scan-AzureAdmins) échouent normalement : ce sont des atomics ciblant un tenant Azure AD, absent de ce lab local.

### 5. Analyse de la détection native

Le dashboard **Threat Hunting** de Wazuh a révélé une couverture de détection native sur plusieurs tactiques MITRE, sans configuration supplémentaire :

| Tactique MITRE ATT&CK | Origine |
|---|---|
| Windows Command Shell | Exécution de `systeminfo` via `cmd.exe` (test Atomic T1082-1) |
| Ingress Tool Transfer | Téléchargement du script d'installation d'Atomic Red Team via `IEX (IWR ...)` |
| Sudo and Sudo Caching | Commandes `sudo` exécutées côté manager pendant l'installation/configuration |
| Account Discovery | Sous-tests annexes de la série T1082 |
| Valid Accounts | Authentifications (session Windows / dashboard) |

<p align="center">
  <img src="assets/images/19-wazuh-discover-systeminfo-event.png" width="80%" />
</p>
<p align="center">
  <img src="assets/images/22-threat-hunting-overview-1.png" width="45%" />
  <img src="assets/images/23-threat-hunting-overview-2.png" width="45%" />
</p>

### 6. Écriture d'une règle de détection personnalisée

Aucune règle native ne couvrait précisément l'exécution de `systeminfo.exe` avec attribution explicite à T1082. Règle ajoutée dans `local_rules.xml` :

\`\`\`xml
<group name="atomic_red_team,mitre_t1082,">
  <rule id="100002" level="5">
    <if_group>sysmon_event1</if_group>
    <field name="win.eventdata.image" type="pcre2">(?i)\\systeminfo\.exe$</field>
    <options>no_full_log</options>
    <description>System information discovery via systeminfo.exe execution</description>
    <mitre>
      <id>T1082</id>
    </mitre>
  </rule>
</group>
\`\`\`

<p align="center"><img src="assets/images/20-local-rules-custom-rule.png" width="80%" /></p>

### 7. Validation dans le temps

Requête `rule.id: 100002` sur 24h → **2 hits confirmés**, avec la bonne description et le bon agent. La règle n'est pas un coup de chance ponctuel, elle est reproductible.

<p align="center">
  <img src="assets/images/24-threat-hunting-rule100002-dashboard.png" width="45%" />
  <img src="assets/images/25-threat-hunting-rule100002-events.png" width="45%" />
</p>

---

## 📊 Résultats

- Agent Windows actif et remontant ses logs en continu.
- Détection native confirmée sur **6 tactiques MITRE ATT&CK** différentes.
- **1 règle de détection personnalisée** écrite, mappée MITRE, validée dans la durée.
- Compréhension fine de la chaîne Wazuh : *decoder → champs extraits → rule → matching → alerte*.

<p align="center"><img src="assets/images/26-endpoint-detail-mitre-tactics.png" width="80%" /></p>

---

## 📚 Ce que ce projet m'a appris

- La différence entre "avoir un SIEM qui tourne" et "avoir un SIEM qui détecte quelque chose de précis" — la seconde partie demande de comprendre la structure des logs, pas seulement l'interface.
- Lire une erreur PowerShell/Bash et distinguer un vrai problème de configuration d'un test qui ne s'applique simplement pas à l'environnement (ex : tests Atomic liés à Azure AD sans tenant Azure).
- L'importance de valider une détection dans le temps, pas juste au moment du test.

---

## 🔭 Pistes d'amélioration

- [ ] Étendre la couverture à d'autres tactiques (Credential Access, Persistence, Lateral Movement)
- [ ] Construire une matrice de couverture complète (technique testée → détectée nativement / via règle custom / non détectée)
- [ ] Ajouter une réponse automatisée (Active Response Wazuh) sur certaines alertes critiques
- [ ] Documenter la gestion des faux positifs

---

## 👤 Auteur

**Yoboue Dje Nicolas** — Étudiant en Mastère Expert IT (cybersécurité, réseaux, systèmes), à la recherche d'une alternance SOC/Blue Team.

- LinkedIn : [linkedin.com/in/yoboue-dje](https://linkedin.com/in/yoboue-dje)
- GitHub : [github.com/djeyoboue44-glitch](https://github.com/djeyoboue44-glitch)
- Portfolio : [djeyoboue44-glitch.github.io](https://djeyoboue44-glitch.github.io)
