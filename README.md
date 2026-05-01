# lab2_sec
# Rapport de Laboratoire : Rooting Android

## 1. Fiche Périmètre

- **Auteur :** HAJAR CHATBAOUI
- **Support :** Android Studio AVD — Pixel 5 API 30
- **Version OS :** Android 11 / API 30
- **Application testée :** F-Droid (org.fdroid.fdroid)
- **Objectif :** Comprendre le rooting, ses impacts sur le sandboxing et les mécanismes Verified Boot / AVB
- **Données / Réseau :** Données fictives uniquement sur réseau local isolé

---

## 2. Fondamentaux Théoriques

### Le Rooting

Le rooting consiste à obtenir les privilèges super-utilisateur (UID 0) sur Android. Cela modifie la confiance du système et permet de briser le sandboxing (isolation des apps). En laboratoire, c'est un outil utile pour inspecter des zones normalement inaccessibles, mais il supprime les garanties d'intégrité logicielle. Il nécessite un environnement isolé, une traçabilité complète et un reset en fin de session.

### Verified Boot & AVB

- **Verified Boot :** garantit que le code exécuté provient d'une source de confiance (fabricant) et n'a pas été altéré.
- **AVB (Android Verified Boot 2.0) :** version moderne qui ajoute une protection contre le rollback (empêche de réinstaller d'anciennes versions vulnérables) et vérifie l'intégrité des partitions system, vendor et boot.
- **Chain of Trust :** chaque composant (bootloader, kernel, system) vérifie la signature du suivant avant de l'exécuter.

---

## 3. Réalisation Technique

### A — Démarrage de l'AVD

L'AVD Pixel 5 API 30 a été lancé depuis Android Studio avec l'option `-writable-system` pour permettre la modification des partitions normalement en lecture seule.

```bash
emulator -avd Pixel_5 -writable-system
```

![AVD démarré](captures/img1.jpg)

### B — Connexion ADB et élévation de privilèges

```bash
adb devices
adb root
adb remount
```

`adb devices` confirme la détection de l'émulateur. `adb root` démarre le serveur ADB en mode root. `adb remount` remonte les partitions système en lecture/écriture — verity est désactivé automatiquement.

![adb devices](captures/img2.jpg)

![adb root + remount](captures/img3.jpg)

### C — Vérification de l'état du système

```bash
adb shell id
adb shell getprop ro.boot.verifiedbootstate
adb shell getprop ro.boot.veritymode
adb shell "su -c id"
```

Résultats obtenus :

| Commande | Résultat | Interprétation |
|----------|----------|----------------|
| `adb shell id` | `uid=0(root)` | Privilèges root confirmés |
| `verifiedbootstate` | `orange` | Intégrité non garantie, système modifié |
| `veritymode` | *(vide)* | Verity désactivé via `-writable-system` |
| `su -c id` | `su: invalid uid/gid` | Normal — le shell est déjà root |

![vérifications système](captures/img4.jpg)

Le résultat `orange` confirme que Verified Boot a bien détecté la modification du système. C'est le comportement attendu et il démontre concrètement l'efficacité du mécanisme AVB.

### D — Installation de F-Droid

```bash
adb install "C:\Users\WINDOWS 11\Downloads\F-Droid.apk"
```

![F-Droid installé](captures/img5.jpg)

### E — Accès aux données privées

```bash
adb shell ls -la /data/data/org.fdroid.fdroid/
```

Cet accès est normalement impossible sans root. Il démontre que le sandboxing Android peut être contourné par un attaquant disposant de privilèges élevés.

![accès /data/data/](captures/img6.jpg)

### F — Capture des logs système

```bash
adb logcat -d > logcat_root_check.txt
dir logcat_root_check.txt
```

Fichier créé : `logcat_root_check.txt` — 2 325 754 octets.

![logcat](captures/img7.jpg)

### G — Remise à zéro

```bash
adb emu avd stop
```

Suivi d'un Wipe Data via Android Studio → Device Manager → Wipe Data sur Pixel_5.

![reset](captures/img8.jpg)

---

## 4. Référentiel OWASP MASVS & MASTG

| Référence | Exigence / Test | Application dans ce lab |
|-----------|-----------------|------------------------|
| MASVS-STORAGE-1 | Les données sensibles doivent être stockées avec chiffrement | Le root permet d'accéder à `/data/data/` pour vérifier si les tokens sont chiffrés ou en clair |
| MASVS-NETWORK-1 | Les communications doivent utiliser TLS correctement configuré | Root permet d'installer des CA système pour intercepter le trafic HTTPS |
| MASTG-TEST-1 | Analyse des SharedPreferences | Inspection de `/data/data/[package]/shared_prefs/` pour détecter des données sensibles en clair |
| MASTG-TEST-2 | Analyse des fuites via Logcat | `adb logcat` pour détecter des informations sensibles pendant l'exécution |

---

## 5. Matrice des Risques & Mesures Défensives

| Risque | Mesure |
|--------|--------|
| Intégrité non garantie → conclusions biaisées | AVD propre dédié aux tests |
| Surface d'attaque accrue | Réseau totalement isolé |
| Données sensibles exposées | Données fictives uniquement, aucun compte personnel |
| Instabilité système | Utilisation de snapshots |
| Mélange comptes perso/test | Aucun compte Google sur l'AVD |
| Mauvais nettoyage fin de séance | Wipe Data systématique |
| Réseau non isolé | Configuration Host-Only |
| Traçabilité insuffisante | Journalisation logcat + captures horodatées |

---
