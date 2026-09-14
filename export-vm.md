# Procédure d'export des VM — Projet Holodeck

Cette procédure explique comment exporter les 2 VM du projet (Holodeck-Serveur et
Holodeck-Client) depuis VMware Workstation, pour pouvoir les transporter ou les
réimporter ailleurs.

## Prérequis

- Les 2 VM doivent être **éteintes** (arrêt propre, pas suspendues) avant l'export
- Prévoir un espace disque suffisant à destination (plusieurs Go par VM — compter
  large, le serveur a un disque de 32 Go et le client de 16 Go)

## Étapes

### 1. Éteindre la VM

Depuis un terminal sur la VM (ou le menu d'extinction pour le client) :
```bash
shutdown now
```
Vérifier dans VMware Workstation que l'état affiché est bien **Powered Off**.

### 2. Vérifier qu'aucun lecteur ISO n'est "bloqué"

⚠️ **Piège rencontré** : l'export a échoué une première fois avec l'erreur :
```
Export failed: File (/home/nouha/Téléchargements/debian-13.4.0-amd64-netinst.iso)
could not be found
```
La VM référençait encore une ancienne image ISO qui avait été supprimée du disque
entre-temps. Solution :
1. **Edit virtual machine settings**
2. Sélectionner **CD/DVD (SATA)**
3. Décocher **Connect at power on**
4. Relancer l'export

### 3. Lancer l'export

Dans VMware Workstation, sélectionner la VM dans la Library, puis :
```
File → Export to OVF...
```
Choisir un dossier de destination avec assez d'espace libre, garder le nom de
fichier proposé par défaut, valider.

⏳ L'export peut prendre plusieurs minutes selon la taille du disque. Ne pas
lancer les deux exports en parallèle — ça ralentit fortement l'opération et
risque de la faire échouer (forte sollicitation disque).

### 4. Vérifier les fichiers générés

Chaque export produit un dossier contenant :

| Fichier | Rôle |
|---------|------|
| `<nom>.ovf` | Définition de la VM (matériel, configuration) |
| `<nom>.mf` | Fichier de manifeste (checksums de vérification) |
| `<nom>-disk1.vmdk` | Le disque virtuel de la VM |

> Pour la VM Cliente, un fichier `.iso` supplémentaire a été inclus dans l'export
> (`Holodeck-Client-file1.iso`) — il correspond à l'image Debian qui était encore
> insérée dans le lecteur CD/DVD virtuel au moment de l'export. Ce n'est pas une
> erreur, juste une inclusion automatique du média monté.

Ne jamais déplacer ou renommer ces fichiers séparément — ils doivent rester
ensemble dans le même dossier pour être réimportables.

### 5. Réimporter une VM exportée (si besoin)

Sur une autre machine avec VMware installé :
```
File → Open...
```
puis sélectionner le fichier `.ovf` — VMware reconstruit automatiquement la VM
à partir des fichiers `.ovf` + `.vmdk` présents dans le même dossier.

## Résultat

| VM | Fichiers exportés | Espace disponible à destination |
|----|--------------------|-----------------------------------|
| Holodeck-Serveur | .ovf, .mf, .vmdk | 190 Go |
| Holodeck-Client | .ovf, .mf, .vmdk, .iso | 190 Go |
