# 🛡️ edr-detection-engineering

> MITRE ATT&CK detection lab Wazuh SIEM/EDR, Sysmon telemetry, adversary emulation with Atomic Red Team, and custom detection rule engineering.

![Status](https://img.shields.io/badge/status-in%20progress-orange)
![Wazuh](https://img.shields.io/badge/Wazuh-4.13.1-blue)
![MITRE ATT%26CK](https://img.shields.io/badge/MITRE-ATT%26CK-red)

---

## 🎯 Objectif du projet

Reproduire, dans un environnement isolé, le cycle complet d'un analyste SOC / détection engineer :
1. Déployer un SIEM/EDR (Wazuh) capable d'ingérer et d'analyser des logs Windows en temps réel.
2. Instrumenter un poste cible avec une télémétrie de qualité (Sysmon).
3. Simuler des techniques d'attaque réelles et documentées (Atomic Red Team, mappées MITRE ATT&CK).
4. Vérifier ce que le SIEM détecte nativement, identifier les angles morts, et **écrire des règles de détection personnalisées** pour combler les manques.
5. Valider chaque règle dans la durée, pas seulement au moment du test.
6. Mesurer les limites (faux positifs, contournements) et les traiter.

Le projet avance par phases :

| Phase | Période | Contenu | Statut |
|---|---|---|---|
| 1 | 07/09 – 08/09/2026 | Déploiement du lab, T1082 (System Information Discovery), règle 100002 | ✅ Terminée |
| 2 | 25/09/2026 | T1003.001 (dump LSASS), règle 100003, analyse du bruit de la règle native 92900 | ✅ Terminée |
| 3 | En cours | Élargissement de la couverture, matrice de couverture, réglage des faux positifs | 🔄 En cours |

Projet réalisé seul, dans le cadre de mon Mastère Expert IT (cybersécurité) et de ma préparation à une alternance SOC / Blue Team.

---

## 🏗️ Architecture du lab

```
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
```

- **Hyperviseur** : VMware Workstation (bascule depuis VirtualBox en cours de route)
- **VM SIEM** : Ubuntu 26.04.1 LTS — 8 vCPU / ~7,2 Go RAM / 49 Go disque
- **VM cible** : Windows 10 Professionnel — 2 vCPU / 4 Go RAM
- Réseau NAT, isolé de l'hôte

---

## 🔧 Stack technique

| Composant | Rôle |
|---|---|
| **Wazuh 4.13.1** (Indexer, Manager, Dashboard) | SIEM/EDR — collecte, corrélation, alerting |
| **Sysmon v15.21** (config [SwiftOnSecurity](https://github.com/SwiftOnSecurity/sysmon-config)) | Télémétrie Windows haute-fidélité (process creation, accès aux processus, réseau, fichiers) |
| **Atomic Red Team** (Invoke-AtomicRedTeam) | Simulation d'attaques mappées MITRE ATT&CK |
| **PowerShell / Bash** | Automatisation et exploitation des logs |
| **Git / GitHub** | Versionnement des règles de détection (`rules/local_rules.xml`) |

---

## 🧪 Phase 1 — Déploiement du lab et première règle (T1082)

### 1. Déploiement du SIEM

Installation de Wazuh en mode *all-in-one* plutôt qu'une stack ELK montée à la main, pour bénéficier des règles de détection MITRE ATT&CK déjà intégrées. Premier disque de 20 Go insuffisant → redimensionné à 49 Go.

<p align="center">
  <img src="01-wazuh-install-log-1.png" width="45%" />
  <img src="02-wazuh-install-log-2.png" width="45%" />
</p>
<p align="center"><img src="03-wazuh-login.png" width="60%" /></p>

### 2. Déploiement de l'agent

Agent Wazuh installé sur la VM Windows (`WIN10-TARGET`), enregistré et actif dans le Dashboard.

<p align="center">
  <img src="04-wazuh-dashboard-no-agents.png" width="45%" />
  <img src="05-deploy-agent-windows.png" width="45%" />
</p>
<p align="center">
  <img src="06-agent-install-powershell.png" width="45%" />
  <img src="07-wazuh-dashboard-agent-active.png" width="45%" />
</p>
<p align="center"><img src="08-endpoints-agent-active.png" width="80%" /></p>

### 3. Instrumentation avec Sysmon

Installation de Sysmon avec la configuration communautaire SwiftOnSecurity (référence du secteur), pour obtenir une télémétrie exploitable (process creation avec ligne de commande complète, notamment).

<p align="center">
  <img src="09-sysmon-download-page.png" width="45%" />
  <img src="10-sysmonconfig-swiftonsecurity.png" width="45%" />
</p>
<p align="center">
  <img src="11-sysmon-install-output.png" width="45%" />
  <img src="12-sysmon-events-check.png" width="45%" />
</p>
<p align="center"><img src="13-wazuh-discover-sysmon-events.png" width="80%" /></p>

### 4. Simulation d'attaque — Atomic Red Team

Installation d'Invoke-AtomicRedTeam et exécution de plusieurs sous-techniques de **T1082 — System Information Discovery** (`systeminfo`, requêtes registre, WMIC, découverte de comptes...).

<p align="center">
  <img src="14-atomicredteam-install-start.png" width="45%" />
  <img src="15-atomicredteam-install-complete.png" width="45%" />
</p>
<p align="center"><img src="16-invoke-atomictest-t1082-list.png" width="80%" /></p>
<p align="center">
  <img src="17-invoke-atomictest-t1082-1-output.png" width="45%" />
  <img src="18-invoke-atomictest-t1082-1-done.png" width="45%" />
</p>
<p align="center"><img src="21-atomictest-t1082-series-run.png" width="80%" /></p>

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
  <img src="19-wazuh-discover-systeminfo-event.png" width="80%" />
</p>
<p align="center">
  <img src="22-threat-hunting-overview-1.png" width="45%" />
  <img src="23-threat-hunting-overview-2.png" width="45%" />
</p>

### 6. Écriture d'une règle de détection personnalisée (100002)

Aucune règle native ne couvrait précisément l'exécution de `systeminfo.exe` avec attribution explicite à T1082. Règle ajoutée dans `local_rules.xml` :

```xml
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
```

<p align="center"><img src="20-local-rules-custom-rule.png" width="80%" /></p>

### 7. Validation dans le temps

Requête `rule.id: 100002` sur 24h → **2 hits confirmés**, avec la bonne description et le bon agent. La règle n'est pas un coup de chance ponctuel, elle est reproductible.

<p align="center">
  <img src="24-threat-hunting-rule100002-dashboard.png" width="45%" />
  <img src="25-threat-hunting-rule100002-events.png" width="45%" />
</p>

---

## 🔐 Phase 2 — Credential Access : dump de LSASS (T1003.001)

### 8. Simulation : dump de la mémoire de LSASS via comsvcs.dll

**T1003.001 — OS Credential Dumping: LSASS Memory** est une technique très utilisée pour voler des identifiants hors ligne. Le test Atomic **T1003.001-2** utilise une DLL native de Windows, sans outil externe :

```powershell
rundll32.exe C:\Windows\System32\comsvcs.dll, MiniDump (Get-Process lsass).id $env:TEMP\lsass-comsvcs.dmp full
```

<p align="center">
  <img src="27-atomictest-t1003-001-list-details.png" width="80%" />
</p>

**Résultat côté cible :** Microsoft Defender bloque l'opération. Le fichier `lsass-comsvcs.dmp` est bien créé, mais il fait **0 octet** : l'attaque échoue, le dump n'est pas exploitable.

<p align="center">
  <img src="28-dump-lsass-defender-blocked-0-bytes.png" width="70%" />
</p>

> **Point important pour un détection engineer :** une attaque bloquée par l'antivirus n'est pas une attaque invisible. Sysmon journalise quand même la *tentative* (Event ID 10, *ProcessAccess*), et c'est cette tentative qu'un SOC veut voir remonter.

### 9. Analyse de la télémétrie et du bruit de la règle native 92900

La requête `agent.name: WIN10-TARGET AND data.win.eventdata.targetImage: *lsass.exe*` fait remonter des événements Sysmon Event ID 10, mais le constat est ambigu : la règle native **92900** (niveau 12, « Lsass process was accessed by … ») se déclenche principalement sur **`MsMpEng.exe`** (Microsoft Defender) et **`svchost.exe`**, c'est-à-dire des accès légitimes.

<p align="center">
  <img src="31-sysmon-event10-msmpeng-lsass.png" width="90%" />
</p>
<p align="center">
  <img src="32-threat-hunting-rule92900-noise.png" width="90%" />
</p>

Conséquences :
- La règle native donne le **même niveau d'alerte (12)** à une activité légitime qu'à une attaque potentielle : c'est du bruit qui noie le signal.
- Elle n'identifie pas la technique utilisée (ici `rundll32` + `comsvcs.dll`).

### 10. Règle personnalisée 100003 — LSASS dump via comsvcs.dll

Règle plus précise, qui croise trois champs de l'événement Sysmon Event ID 10 : la cible (`lsass.exe`), la source (`rundll32.exe`) et la présence de `comsvcs.dll` dans la pile d'appels (`callTrace`) :

```xml
<group name="sysmon,">
  <rule id="100003" level="12">
    <if_group>sysmon_event_10</if_group>
    <field name="win.eventdata.targetImage" type="pcre2">(?i)lsass\.exe$</field>
    <field name="win.eventdata.sourceImage" type="pcre2">(?i)rundll32\.exe$</field>
    <field name="win.eventdata.callTrace" type="pcre2">(?i)comsvcs\.dll</field>
    <description>T1003.001 - LSASS dump via comsvcs.dll (rundll32)</description>
    <mitre>
      <id>T1003.001</id>
    </mitre>
    <group>atomic_red_team,credential_access,mitre_t1003</group>
  </rule>
</group>
```

Une étape de débogage a été nécessaire : le nom du groupe parent devait correspondre exactement à celui utilisé dans le ruleset Wazuh. Je l'ai vérifié en comparant avec `0800-sysmon_id_1.xml`, puis corrigé `sysmon_event10` → `sysmon_event_10` avec `sed`.

<p align="center">
  <img src="35-local-rules-100003-sed-fix.png" width="90%" />
</p>

### 11. Validation

Requête `100003` dans Threat Hunting → **3 hits** (25/09/2026, 16:37 et 16:40), niveau 12, description « T1003.001 - LSASS dump via comsvcs.dll (rundll32) », agent `WIN10-TARGET`.

<p align="center">
  <img src="37-threat-hunting-rule100003-3-hits.png" width="90%" />
</p>

---

## 📊 Résultats

- Agent Windows actif et remontant ses logs en continu.
- Détection native confirmée sur **6 tactiques MITRE ATT&CK** différentes (phase 1).
- **2 règles de détection personnalisées** écrites, mappées MITRE et validées (100002 pour T1082, 100003 pour T1003.001).
- Tentative de dump LSASS **détectée même quand elle est bloquée** par Defender (dump de 0 octet).
- Bruit de la règle native 92900 identifié et documenté (MsMpEng.exe, svchost.exe).
- Compréhension fine de la chaîne Wazuh : *decoder → champs extraits → rule → matching → alerte*.
- Règles versionnées dans le dépôt : [`rules/local_rules.xml`](rules/local_rules.xml).

<p align="center"><img src="26-endpoint-detail-mitre-tactics.png" width="80%" /></p>

### Limites connues

- La règle **100003 est volontairement étroite** : elle ne couvre que `rundll32` + `comsvcs.dll`. Les autres méthodes du même T1003.001 (ProcDump, NanoDump, Mimikatz, `createdump.exe`, etc.) ne sont pas encore détectées par une règle personnalisée.
- Le bruit de la règle native 92900 n'est pas encore filtré.

---

## 📚 Ce que ce projet m'a appris

- La différence entre "avoir un SIEM qui tourne" et "avoir un SIEM qui détecte quelque chose de précis" — la seconde partie demande de comprendre la structure des logs, pas seulement l'interface.
- Lire une erreur PowerShell/Bash et distinguer un vrai problème de configuration d'un test qui ne s'applique simplement pas à l'environnement (ex : tests Atomic liés à Azure AD sans tenant Azure).
- L'importance de valider une détection dans le temps, pas juste au moment du test.
- Qu'un antivirus qui bloque une attaque ne dispense pas de la détecter : la tentative reste un indicateur précieux pour un SOC.
- Qu'une règle native peut être trop générique et produire des faux positifs : détecter, c'est aussi savoir distinguer l'activité légitime de l'activité suspecte.

---

## 🔭 Feuille de route

- [x] Déployer Wazuh + Sysmon + Atomic Red Team
- [x] Règle personnalisée T1082 (100002)
- [x] Règle personnalisée T1003.001 via comsvcs.dll (100003)
- [ ] Recréer et documenter une règle pour T1218.011 (Rundll32)
- [ ] Filtrer le bruit de la règle native 92900 (sources légitimes : MsMpEng.exe, svchost.exe)
- [ ] Généraliser la détection LSASS (masques d'accès `GrantedAccess`, autres outils)
- [ ] Étendre la couverture à d'autres tactiques (Persistence, Execution, Defense Evasion, Lateral Movement)
- [ ] Construire une matrice de couverture complète (technique testée → détectée nativement / via règle custom / non détectée)
- [ ] Ajouter une détection comportementale (baseline, anomalies) et une couche d'analyse assistée par IA
- [ ] Ajouter une réponse automatisée (Active Response Wazuh) sur certaines alertes critiques

---

## 👤 Auteur

**Yoboue Dje Nicolas** — Étudiant en Mastère Expert IT (cybersécurité, réseaux, systèmes), à la recherche d'une alternance SOC/Blue Team.

- LinkedIn : [linkedin.com/in/yoboue-dje](https://linkedin.com/in/yoboue-dje)
- GitHub : [github.com/djeyoboue44-glitch](https://github.com/djeyoboue44-glitch)
- Portfolio : [djeyoboue44-glitch.github.io](https://djeyoboue44-glitch.github.io)
