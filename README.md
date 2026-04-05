# KB-Manager v2.0
### Gestionnaire Intelligent de Mises à Jour Windows

---

## Présentation

KB-Manager est un outil professionnel pour gérer les mises à jour Windows identifiées par leur numéro KB. Il permet de les lister, les bloquer de façon permanente, les désinstaller et surveiller automatiquement les tentatives de réinstallation.

**Contrairement à Hide Update (wushowhide.diagcab)**, le blocage opère sur trois niveaux simultanément et résiste aux Feature Updates et aux correctifs cumulatifs.

---

## Compilation

### Prérequis
- SDK .NET 8 : https://dotnet.microsoft.com/download
- Windows 10 ou 11 x64

### Build standard (requiert .NET 8 sur le PC cible)
```
build.cmd
```
Produit `publish\kb-manager.exe` (~12 Mo)

### Build autonome (zéro dépendance, runtime inclus)
```
build-standalone.cmd
```
Produit `publish-standalone\kb-manager.exe` (~70 Mo, fonctionne sur tout Windows 10/11)

---

## Utilisation

### Mode Interface Graphique
```
kb-manager.exe
```
Lance l'interface WPF avec dashboard, liste des KB, liste noire, journal.

### Mode Terminal

```
kb-manager.exe --help                      Aide rapide
kb-manager.exe --man                       Manuel complet paginé
kb-manager.exe --man --block               Manuel de la commande --block

kb-manager.exe --list                      Liste les KB installées
kb-manager.exe --list --pending            KB en attente
kb-manager.exe --list --blocked            KB bloquées
kb-manager.exe --list --output json        Sortie JSON

kb-manager.exe --block KB5034441           Bloque une KB
kb-manager.exe --block KB5034441 KB5032190 Bloc multiple
kb-manager.exe --unblock KB5034441         Débloque une KB
kb-manager.exe --remove KB5034441          Désinstalle + bloque

kb-manager.exe --status KB5034441          Statut détaillé
kb-manager.exe --watchdog install          Installe le watchdog
kb-manager.exe --watchdog remove           Supprime le watchdog
kb-manager.exe --watchdog status           État du watchdog
kb-manager.exe --watchdog run              Déclenche le watchdog

kb-manager.exe --export blocklist.json     Exporte la liste noire
kb-manager.exe --import blocklist.json     Importe une liste noire
kb-manager.exe --log                       20 derniers événements
kb-manager.exe --log --tail 100            100 derniers événements
```

### Options globales
```
--output table|json|csv    Format de sortie
--no-color                 Pas de couleurs ANSI
--quiet                    Messages minimaux
--verbose                  Détail des opérations
```

---

## Comment fonctionne le blocage

Le blocage d'une KB agit simultanément sur trois niveaux :

**1. Registre Windows (niveau GPO)**
Écriture dans `HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate`. C'est la même méthode qu'un administrateur réseau en entreprise via Group Policy. Ce blocage persiste après redémarrage.

**2. Windows Update Agent (WUA)**
Marquage de la KB comme "cachée" via l'API COM officielle de Windows Update. Identique à ce que fait Hide Update, mais en plus robuste.

**3. Liste noire JSON persistante**
Stockée dans `%APPDATA%\KB-Manager\blocklist.json`. Surveillée en permanence par le watchdog.

---

## Le Watchdog

Le watchdog est une tâche planifiée Windows qui re-applique automatiquement tous les blocages sans intervention manuelle. Il se déclenche dans quatre situations :

- Au démarrage de Windows (30 secondes après le boot)
- Quand le service Windows Update démarre
- Après chaque installation de mise à jour détectée
- Toutes les heures (filet de sécurité)

Si Windows tente de réinstaller une KB bloquée, le watchdog la re-bloque immédiatement et envoie une alerte.

**Installation :**
```
kb-manager.exe --watchdog install
```

---

## Surveillance temps réel (mode GUI)

En mode interface graphique, KB-Manager surveille en temps réel via WMI les événements Windows Update. Si une KB de la liste noire est détectée :
- Une fenêtre d'alerte toast apparaît en bas à droite de l'écran
- Le blocage est re-appliqué immédiatement
- L'événement est enregistré dans le journal

---

## Codes de sortie (mode CLI)

| Code | Signification |
|------|---------------|
| 0 | Succès |
| 1 | Erreur générique |
| 2 | KB introuvable |
| 3 | Droits administrateur insuffisants |
| 4 | KB déjà bloquée |
| 5 | KB déjà désinstallée |
| 6 | Watchdog non installé |
| 10 | Erreur API Windows Update |

**Exemple dans un script PowerShell :**
```powershell
kb-manager.exe --block KB5034441
switch ($LASTEXITCODE) {
    0 { Write-Host "Bloqué avec succès" -ForegroundColor Green }
    3 { Write-Host "Relancer en administrateur" -ForegroundColor Red }
    4 { Write-Host "Déjà dans la liste noire" -ForegroundColor Yellow }
}
```

---

## Fichiers de données

| Fichier | Contenu |
|---------|---------|
| `%APPDATA%\KB-Manager\blocklist.json` | Liste noire des KB bloquées |
| `%APPDATA%\KB-Manager\kb-manager.log.json` | Journal des événements |
| `%APPDATA%\KB-Manager\settings.json` | Paramètres de l'outil |

---

## Droits nécessaires

| Opération | Droits requis |
|-----------|--------------|
| Lister les KB installées | Utilisateur standard |
| Lister les KB en attente | Utilisateur standard |
| Bloquer (registre GPO) | **Administrateur** |
| Désinstaller une KB | **Administrateur** |
| Installer le watchdog | **Administrateur** |
| Lire les logs | Utilisateur standard |

L'interface graphique détecte automatiquement si les droits sont insuffisants et propose de relancer en administrateur.

---

## Déploiement multi-postes

Pour déployer la même liste noire sur plusieurs machines :

```cmd
:: Sur le poste de référence
kb-manager.exe --export \\serveur\partage\blocklist.json

:: Sur chaque poste cible
kb-manager.exe --import \\serveur\partage\blocklist.json
kb-manager.exe --watchdog install
```

---

## Structure du projet

```
KB-Manager/
├── App.xaml / App.xaml.cs          Point d'entrée dual-mode GUI/CLI
├── KB-Manager.csproj               Configuration build .NET 8
├── build.cmd                       Build standard
├── build-standalone.cmd            Build autonome (runtime inclus)
├── Properties/
│   ├── AssemblyInfo.cs             Métadonnées assemblage
│   └── app.manifest                UAC + DPI awareness
├── Models/
│   └── Models.cs                   KbUpdate, BlockRule, AppSettings, LogEntry
├── Core/
│   ├── Logger.cs                   Journal JSON horodaté
│   ├── KbDatabase.cs               Persistance blocklist + settings
│   ├── WindowsUpdateEngine.cs      API WMI + COM WUA
│   ├── RegistryBlocker.cs          Blocage registre niveau GPO
│   ├── BlockOrchestrator.cs        Orchestration des 3 niveaux
│   ├── TaskSchedulerService.cs     Watchdog tâche planifiée
│   ├── WuEventMonitor.cs           Surveillance temps réel WMI
│   ├── AlertService.cs             Notifications + tray icon
│   └── ElevationService.cs         Gestion droits UAC
└── UI/
    ├── MainWindow.xaml/.cs         Dashboard principal
    ├── KbDetailWindow.xaml/.cs     Détails d'une KB
    ├── AlertWindow.xaml/.cs        Toast d'alerte bloquage
    └── SettingsWindow.xaml/.cs     Paramètres
```

---

## Notes importantes

- Les Feature Updates majeurs (passage 22H2 → 23H2 par exemple) peuvent contourner certains blocages registre. Le watchdog re-applique les règles automatiquement après chaque redémarrage.
- Le numéro KB peut être saisi avec ou sans le préfixe `KB` en mode CLI.
- Le blocage via registre GPO est identique à ce qu'utilise un administrateur réseau en entreprise. C'est la méthode la plus robuste disponible.

---

**KB-Manager v2.0** — C# .NET 8 WPF
