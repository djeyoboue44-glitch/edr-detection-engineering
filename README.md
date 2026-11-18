# 🛡️ edr-detection-engineering

> MITRE ATT&CK detection lab — Wazuh SIEM/EDR, Sysmon telemetry, adversary emulation with Atomic Red Team, and custom detection rule engineering.

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
| 3 | octobre 2026 | T1218.011 (Rundll32), T1547.001 (persistance), T1490 (suppression des sauvegardes), règles 100004, 100006, 100007, 100008, matrice de couverture | ✅ Terminée |
| 4 | 09/11/2026 | Réglage du bruit des règles natives : 92900 (Defender) et 92213 (663 alertes de niveau 15), règles 100005, 100009, 100010 | ✅ Terminée |
| 5 | À venir | Accès à LSASS par `svchost.exe`, mesure des faux positifs dans la durée, couche d'analyse assistée par IA | ⏳ Prévue |

Projet réalisé seul, dans le cadre de mon Mastère Expert IT (cybersécurité), en parallèle de mon alternance.

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
<p align="center"><img src="30-sysmon-config-export.PNG" width="80%" /></p>

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
  <img src="27-atomictest-t1003-001-list-details.PNG" width="80%" />
</p>
<p align="center">
  <img src="38-atomictest-t1003-001-2-repeated-runs.PNG" width="80%" />
</p>

**Résultat côté cible :** Microsoft Defender bloque l'opération. Le fichier `lsass-comsvcs.dmp` est bien créé, mais il fait **0 octet** : l'attaque échoue, le dump n'est pas exploitable.

<p align="center">
  <img src="28-dump-lsass-defender-blocked-0-bytes.PNG" width="70%" />
</p>

> **Point important pour un détection engineer :** une attaque bloquée par l'antivirus n'est pas une attaque invisible. Sysmon journalise quand même la *tentative* (Event ID 10, *ProcessAccess*), et c'est cette tentative qu'un SOC veut voir remonter.

### 9. Analyse de la télémétrie et du bruit de la règle native 92900

La requête `agent.name: WIN10-TARGET AND data.win.eventdata.targetImage: *lsass.exe*` fait remonter des événements Sysmon Event ID 10, mais le constat est ambigu : la règle native **92900** (niveau 12, « Lsass process was accessed by … ») se déclenche principalement sur **`MsMpEng.exe`** (Microsoft Defender) et **`svchost.exe`**, c'est-à-dire des accès légitimes.

<p align="center">
  <img src="31-sysmon-event10-msmpeng-lsass.PNG" width="90%" />
</p>
<p align="center">
  <img src="32-threat-hunting-rule92900-noise.PNG" width="90%" />
</p>
<p align="center">
  <img src="33-threat-hunting-events-24h.PNG" width="45%" />
  <img src="34-threat-hunting-dashboard-24h.PNG" width="45%" />
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
  <img src="35-local-rules-100003-sed-fix.PNG" width="90%" />
</p>

### 11. Validation

Requête `100003` dans Threat Hunting → **3 hits** (25/09/2026, 16:37 et 16:40), niveau 12, description « T1003.001 - LSASS dump via comsvcs.dll (rundll32) », agent `WIN10-TARGET`.

<p align="center">
  <img src="37-threat-hunting-rule100003-3-hits.PNG" width="90%" />
</p>

---

## 🧩 Phase 3 — Élargir la couverture : exécution, persistance, impact (octobre 2026)

Trois nouvelles techniques, même démarche à chaque fois : simuler avec Atomic Red Team, regarder ce que Wazuh dit nativement, identifier l'angle mort, écrire une règle sur ce que le SIEM **reçoit réellement**, puis valider avec une vraie alerte.

> Principe retenu : **détecter l'effet, pas l'outil**. Les règles natives surveillent souvent un exécutable précis (`reg.exe`, `rundll32.exe`) alors qu'un attaquant peut obtenir le même résultat autrement.

### 12. T1218.011 — Rundll32 : exécution d'un fichier qui n'est pas une DLL

`rundll32.exe` charge normalement des `.dll` (ou `.cpl`, `.ocx`). Les tests Atomic **-10** (`rundll32.exe calc.png, StartW`) et **-14** (`rundll32.exe _WT.init,krnl`) lui font charger un fichier d'une autre extension, une technique d'évasion courante.

- **Trace Sysmon :** Event ID 1 (création de processus), `rundll32.exe` avec un premier argument non-DLL.
- **Détection native :** aucune alerte, l'événement est reçu puis ignoré.
- **Règle 100004 (niveau 10)** : expression régulière sur la ligne de commande qui repère un premier argument dont l'extension n'est pas `dll`, `cpl` ou `ocx`.

Une première version de l'expression ne se déclenchait jamais : **la ligne de commande telle que Windows l'affiche n'est pas celle que Wazuh reçoit** (barres obliques inverses doublées, guillemets échappés). La version finale tolère une barre oblique facultative avant les guillemets.

**Validation :** 5 alertes (3 pour `calc.png`, 2 pour `_WT.init`) le 12/10/2026.
**Limite :** une DLL malveillante portant une extension `.dll` passe inaperçue ; les variantes JavaScript et VBScript (tests -1, -2, -3) ne sont pas couvertes.

### 13. T1547.001 — Persistance par clé de registre Run

Sysmon journalise l'écriture d'une valeur dans `Run` / `RunOnce` (Event ID 13). Côté Wazuh :

- la règle native **92302** (niveau 6) ne voit que `reg.exe` ;
- la règle de base **92300** est au **niveau 0** (silencieuse) ;
- la règle **92041** a sa partie « Base64 » entièrement facultative : elle se déclenche sur n'importe quel `reg add /d`.

**Angle mort :** la même persistance posée par **PowerShell**, `wscript`, `cscript` ou `mshta` ne produit aucune alerte.

**Règle 100006 (niveau 8)** : fille de 92300, elle ne se déclenche que si le processus est `powershell`, `pwsh`, `wscript`, `cscript` ou `mshta`, avec l'étiquette MITRE T1547.001.

**Validation :** alertes à 01:36:34 (PowerShell 64 bits) et 01:46:42 (PowerShell 32 bits, clé `WOW6432Node`) le 21/10/2026.

Ce que ce test a appris :
- une écriture dans `HKLM` demande des **droits administrateur** ;
- un PowerShell **32 bits** écrit dans `WOW6432Node` et `SysWOW64` ;
- le code de sortie (« Exit code ») ne prouve rien : il faut relire la clé ;
- Wazuh journalise `HKU\S-1-5-21…` et non `HKCU`.

**Limite :** liste fixe d'interpréteurs ; le dossier Démarrage (tests -4 à -7) n'est pas testé (2 tests sur 21 réalisés).

### 14. T1490 — Inhibit System Recovery : suppression des clichés instantanés

Un ransomware supprime les copies de sauvegarde Windows avant de chiffrer. Tests Atomic **-1** (`vssadmin delete shadows`) et **-2** (`wmic shadowcopy delete`).

- **Détection native :** aucune règle rattachée à T1490, seulement du bruit sur `cmd.exe` (niveaux 3 et 4).
- **Règles 100007 (vssadmin) et 100008 (wmic), niveau 12** : elles s'appuient sur `originalFileName` (le nom d'origine intégré à l'exécutable) plutôt que sur le nom du fichier, ce qui résiste à un exécutable renommé, et sur la ligne de commande (`delete shadows` / `shadowcopy delete`).
- Les commandes de test contiennent **deux espaces** entre les mots : l'expression utilise `\s+`.

**Validation :** une alerte chacune (19:45:41 pour -1, 00:27:30 pour -2).
**Limite :** aucune copie réelle n'a été supprimée ; les variantes PowerShell (-5) et `diskshadow` (-13) ne sont pas couvertes.

### 15. Matrice de couverture

Synthèse des cinq techniques testées : trace Sysmon, détection native, règle personnalisée, validation, limites. Document complet : [`docs/Matrice_de_couverture_EDR.pdf`](docs/Matrice_de_couverture_EDR.pdf).

| Technique | Tactique | Détection native | Règle personnalisée |
|---|---|---|---|
| T1082 | Discovery | Alertes génériques, pas d'attribution à T1082 | 100002 |
| T1003.001 | Credential Access | 92900 : bruit de Defender, ne voit pas l'attaque | 100003 (+ 100005 réglage) |
| T1218.011 | Defense Evasion | Aucune alerte | 100004 |
| T1547.001 | Persistence | 92302 voit `reg.exe` seulement | 100006 |
| T1490 | Impact | Aucune règle rattachée | 100007, 100008 |

---

## 🔇 Phase 4 — Régler le bruit des règles natives (novembre 2026)

Une détection qui crie tout le temps n'est plus lue. Cette phase s'attaque à deux règles natives trop larges, avec la même méthode : **compter, identifier qui déclenche, écrire une règle étroite, vérifier**.

> Méthode de comptage : uniquement dans les fichiers d'alertes **datés** (pas `alerts.json`, qui ne contient que le jour en cours), avec `grep -a` : sans cette option, `grep` s'arrête sur des lignes qu'il prend pour du binaire et le total est faux.

### 16. Règle 92900 → règle de réglage 100005 (Defender)

La règle native 92900 (accès à LSASS, niveau 12) se déclenche surtout pour `MsMpEng.exe`, le moteur de Microsoft Defender. La règle **100005** (niveau 3, fille de 92900) reclasse les accès dont la source est exactement `MsMpEng.exe` dans le dossier `Platform` de Defender.

Comptage au 09/11/2026 : **81** alertes de 92900 conservées, **48** reclassées par 100005.
**Reste à traiter :** `svchost.exe` continue de déclencher 92900 au niveau 12 (observé le 09/11).

<p align="center">
  <img src="41-threat-hunting-events-svchost-lsass-92900.png" width="90%" />
</p>

### 17. Règle native 92213 : 663 alertes de niveau 15

La règle 92213 se déclenche quand un exécutable ou un script est créé dans un dossier souvent utilisé par des malwares (ici le dossier `Temp`). Sur le poste de test : **663 alertes de niveau 15**.

| Processus qui crée le fichier | Alertes | Ce que c'est |
|---|---|---|
| `cleanmgr.exe` | 396 | Nettoyage de disque Windows, qui recopie les modules DISM dans `Temp` |
| `powershell.exe` (64 bits) | 187 | Dont des fichiers `__PSScriptPolicyTest` |
| `powershell.exe` (32 bits) | 72 | Idem |
| `sdiagnhost.exe` | 4 | Dépanneur Windows |
| `csc.exe` | 2 | Compilateur .NET (fichiers temporaires) |
| `rundll32.exe` (32 bits) | 1 | Non examiné |
| `msedgewebview2.exe` | 1 | Composant Edge |

143 des alertes de PowerShell concernent des fichiers `__PSScriptPolicyTest_xxxx.ps1`, que PowerShell crée lui-même. Les alertes de `cleanmgr` forment une série d'environ 25 fichiers (`DismHost.exe`, `DismCore.dll`, `*Provider.dll`…) qui se répète à chaque passage.

Deux règles de réglage **étroites**, qui exigent **trois conditions à la fois** (processus, dossier, nom de fichier) et ne baissent le niveau que de 15 à 3 :

```xml
<group name="sysmon,tuning,">

  <rule id="100009" level="3">
    <if_sid>92213</if_sid>
    <field name="win.eventdata.image" type="pcre2">(?i)^C:\\\\Windows\\\\system32\\\\cleanmgr\.exe$</field>
    <field name="win.eventdata.targetFilename" type="pcre2">(?i)\\\\AppData\\\\Local\\\\Temp\\\\[^\\]+\\\\(?:Dism[A-Za-z]*|[A-Za-z]*Provider)\.(?:dll|exe)$</field>
    <description>Nettoyage de disque Windows (cleanmgr) copie les modules DISM dans Temp - bruit attendu</description>
  </rule>

  <rule id="100010" level="3">
    <if_sid>92213</if_sid>
    <field name="win.eventdata.image" type="pcre2">(?i)^C:\\\\Windows\\\\(?:System32|SysWOW64)\\\\WindowsPowerShell\\\\v1\.0\\\\powershell\.exe$</field>
    <field name="win.eventdata.targetFilename" type="pcre2">(?i)\\\\AppData\\\\Local\\\\Temp\\\\__PSScriptPolicyTest_[a-z0-9]+\.[a-z0-9]+\.ps1$</field>
    <description>PowerShell crée son fichier de test de politique d'exécution dans Temp - bruit attendu</description>
  </rule>

</group>
```

Fichier : [`rules/tuning-92213-100009-100010.xml`](rules/tuning-92213-100009-100010.xml).

**Pourquoi des règles étroites :** ignorer tout ce que font PowerShell ou `cleanmgr` dans `Temp` permettrait à un attaquant de se cacher derrière ces deux processus. Toute autre DLL ou tout autre exécutable déposé dans `Temp` reste au niveau 15.

**Validation (09/11/2026) :**
- **100010** : alerte de niveau 3 à 23:46:22 sur `WIN10-TARGET`, après un test Atomic Red Team.
- **100009** : 26 alertes de niveau 3 à 23:54, soit un passage de `cleanmgr` (environ 25 fichiers d'un coup).

<p align="center">
  <img src="39-threat-hunting-rule100009-cleanmgr-26-hits.png" width="90%" />
</p>
<p align="center">
  <img src="40-threat-hunting-rule100010-powershell-policytest.png" width="90%" />
</p>

> Dans la capture ci-dessus, les lignes « Successful sudo to ROOT » sont du bruit de la recherche en texte libre (le nombre `100010` apparaît dans mes commandes `sudo`). La recherche propre est `rule.id:100010`.

**Ce qui n'est pas couvert, et reste au niveau 15 :** environ 116 fichiers créés par PowerShell qui ne sont pas des `PSScriptPolicyTest` (non identifiés), un `rundll32` et un `csc` vus une seule fois.

### 18. Erreurs rencontrées et leçons

| Ce qui s'est passé | Leçon |
|---|---|
| Comptages d'alertes gonflés par une double lecture | Compter dans les fichiers datés uniquement |
| `grep` : « fichiers binaires correspondent », total faux (324 au lieu de 663) | Utiliser `-a` et comparer deux totaux indépendants |
| Comptes à zéro | Le dossier des alertes n'est lisible que par root : lancer toute la boucle dans `sudo bash -c` |
| Règles de réglage étiquetées `mitre_t1490` | Un bloc `<group>` donne ses étiquettes à toutes ses règles : ranger les règles de réglage dans un bloc `sysmon,tuning,` |
| Règle en double (même ID) après une commande lancée deux fois | Garde `grep -q` avant ajout, et `wazuh-analysisd -t` avant redémarrage |
| Plus de connexion au tableau de bord après une coupure | Le service DHCP du réseau VMware était désactivé : l'adresse de la VM a changé et l'agent ne la retrouvait plus. Fixer l'adresse de la VM |

---

## 📊 Résultats

- Agent Windows actif et remontant ses logs en continu.
- Détection native confirmée sur **6 tactiques MITRE ATT&CK** différentes (phase 1).
- **5 techniques testées** (T1082, T1003.001, T1218.011, T1547.001, T1490) sur 5 tactiques.
- **9 règles personnalisées** (100002 à 100010) : **6 règles de détection** (100002, 100003, 100004, 100006, 100007, 100008) et **3 règles de réglage** du bruit (100005, 100009, 100010).
- Tentative de dump LSASS **détectée même quand elle est bloquée** par Defender (dump de 0 octet).
- Bruit des règles natives mesuré et en partie réglé : 92900 (81 alertes, dont 48 reclassées) et 92213 (663 alertes de niveau 15, dont l'essentiel expliqué par `cleanmgr.exe` et PowerShell).
- Matrice de couverture : [`docs/Matrice_de_couverture_EDR.pdf`](docs/Matrice_de_couverture_EDR.pdf).
- Règles versionnées dans le dépôt : [`rules/`](rules/).

<p align="center"><img src="26-endpoint-detail-mitre-tactics.png" width="80%" /></p>
<p align="center">
  <img src="29-wazuh-overview-24h.PNG" width="45%" />
  <img src="36-wazuh-overview-alerts.PNG" width="45%" />
</p>

### Limites connues

- Chaque règle de détection est **volontairement étroite** : 100003 ne couvre que `rundll32` + `comsvcs.dll` (pas ProcDump, NanoDump, Mimikatz…), 100006 repose sur une liste fixe d'interpréteurs, 100004 ne voit pas une DLL malveillante nommée `.dll`.
- Validation par **un ou deux tests par technique** (2 sur 21 pour la persistance, 2 sur 12 pour T1490), sur **un seul poste Windows**.
- Les faux positifs de 100004, 100006, 100007 et 100008 n'ont été observés que sur une courte période.
- Le bruit de `svchost.exe` sur la règle 92900 n'est pas encore traité.
- L'effet exact du réglage de 92213 n'est pas encore chiffré sur plusieurs jours (comparaison avant / après).

---

## 📚 Ce que ce projet m'a appris

- La différence entre « avoir un SIEM qui tourne » et « avoir un SIEM qui détecte quelque chose de précis » : la seconde partie demande de comprendre la structure des logs, pas seulement l'interface.
- Lire une erreur PowerShell ou Bash et distinguer un vrai problème de configuration d'un test qui ne s'applique pas à l'environnement (ex : tests Atomic liés à Azure AD sans tenant Azure).
- L'importance de valider une détection dans la durée, pas seulement au moment du test.
- Qu'un antivirus qui bloque une attaque ne dispense pas de la détecter : la tentative reste un indicateur précieux pour un SOC.
- **Détecter l'effet, pas l'outil** : une règle sur `reg.exe` rate PowerShell, une règle sur le nom d'un fichier rate un exécutable renommé (d'où `originalFileName`).
- **Écrire une règle sur ce que le SIEM reçoit, pas sur ce que Windows affiche** (barres obliques doublées, guillemets échappés, `WOW6432Node`).
- **Mesurer avant de régler** : compter, identifier qui déclenche, puis écrire une règle étroite. Un réglage trop large est une faille.
- Qu'un chiffre doit être vérifié par deux méthodes avant d'être publié (comptage tronqué, double lecture).

---

## 🔭 Feuille de route

- [x] Déployer Wazuh + Sysmon + Atomic Red Team
- [x] Règle personnalisée T1082 (100002)
- [x] Règle personnalisée T1003.001 via comsvcs.dll (100003)
- [x] Règle personnalisée T1218.011 (Rundll32, 100004)
- [x] Règle personnalisée T1547.001 (persistance, 100006)
- [x] Règles personnalisées T1490 (100007, 100008)
- [x] Matrice de couverture des 5 techniques testées
- [x] Filtrer le bruit de Defender sur la règle native 92900 (100005)
- [x] Régler le bruit de la règle native 92213 (100009, 100010)
- [ ] Traiter l'accès à LSASS par `svchost.exe` (règle 92900)
- [ ] Identifier les ~116 fichiers PowerShell restants de la règle 92213
- [ ] Mesurer les faux positifs dans la durée et chiffrer l'effet du réglage (avant / après)
- [ ] Généraliser la détection LSASS (masques d'accès `GrantedAccess`, autres outils)
- [ ] Étendre la couverture à d'autres tactiques (Lateral Movement…) et à d'autres tests par technique
- [ ] Ajouter une couche d'analyse assistée par IA qui **propose** des règles, validées par un humain
- [ ] Ajouter une réponse automatisée (Active Response Wazuh) sur certaines alertes critiques

---

## 👤 Auteur

**Yoboue Dje Nicolas** — Étudiant en Mastère Expert IT (cybersécurité, réseaux, systèmes), en alternance.

- LinkedIn : [linkedin.com/in/yoboue-dje](https://linkedin.com/in/yoboue-dje)
- GitHub : [github.com/djeyoboue44-glitch](https://github.com/djeyoboue44-glitch)
- Portfolio : [djeyoboue44-glitch.github.io](https://djeyoboue44-glitch.github.io)
