# Notice d'installation et d'utilisation — Projet Holodeck

Cette notice décrit l'installation complète de l'infrastructure Holodeck :
une VM Serveur Debian (DHCP/DNS/Web/FTP/LDAP) et une VM Cliente Debian avec GUI.

## Prérequis

- **VMware Workstation** installé sur la machine hôte
- Une image ISO **Debian 13.x** (netinst ou DVD complet — DVD recommandé pour la VM Cliente
  si elle n'a pas accès à internet pendant l'installation)
- Au moins 4 Go de RAM disponibles et 48 Go d'espace disque libre (32 Go serveur + 16 Go client)

---

## Partie 1 — VM Serveur

### 1.1 Création de la VM

Dans VMware Workstation, créer une VM avec :
- **Custom (advanced)** au lieu de Typical
- RAM : **2 Go**
- Processeurs : **2 vCPU**
- Disque : **32 Go**
- Réseau : deux cartes réseau
  - **Adapter 1** : NAT ou Bridged (WAN, accès internet pour les dépôts)
  - **Adapter 2** : **Host-only** (LAN, réseau privé pour la VM Cliente)
- Debian **sans interface graphique** (installation minimale)

### 1.2 Configuration réseau

Après installation de base, configurer les deux interfaces dans
`/etc/network/interfaces` :

```
# WAN (DHCP automatique)
auto ens33
iface ens33 inet dhcp

# LAN (IP fixe)
auto ens34
iface ens34 inet static
    address 192.168.100.1
    netmask 255.255.255.0
```

⚠️ **Piège connu** : si le client DHCP (`dhcpcd`) réécrit `/etc/resolv.conf`,
rendre la configuration DNS permanente dans `/etc/dhcpcd.conf` :
```
static domain_name_servers=192.168.100.1 127.0.0.1
```

### 1.3 Utilisateur et sécurité de base

- Créer un utilisateur standard (ex. `nouha`), **ne pas** installer le paquet `sudo`
- Travailler en root via `su -`

### 1.4 Pare-feu (nftables)

Fichier `/etc/nftables.conf` — politique par défaut `drop`, ports autorisés :
22 (SSH), 67 (DHCP), 53 (DNS), 80/443 (HTTP/HTTPS), 21 + 40000-40100 (FTP passif), ICMP.

⚠️ **Piège connu** : vérifier la plage FTP passive (`40000-40100`, pas `4000-40100`).

Activer et charger la configuration :
```bash
nft -f /etc/nftables.conf
systemctl enable nftables
```

### 1.5 DHCP (isc-dhcp-server)

Installer et configurer pour écouter uniquement sur `ens34` :
```
# /etc/default/isc-dhcp-server
INTERFACESv4="ens34"
```

Plage `192.168.100.10` à `192.168.100.100`, domaine `starfleet.lan`.

⚠️ **Piège connu** : l'option se nomme `option routers` (anglais), pas `option routeurs`.

Toujours valider avant de recharger :
```bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf
systemctl restart isc-dhcp-server
```

### 1.6 DNS (bind9)

Zone `starfleet.lan` dans `/etc/bind/db.starfleet.lan`, déclarée dans
`/etc/bind/named.conf.local`. Enregistrements A pour `www7`, `www8`, `php`,
`admin`, `_2admin` → tous vers `192.168.100.1`.

Ajouter des **forwarders** dans `/etc/bind/named.conf.options` (ex. 8.8.8.8, 1.1.1.1)
pour conserver l'accès internet une fois le serveur configuré comme DNS.

⚠️ **Pièges connus** :
- `_2admin` (underscore) refusé par défaut → ajouter `check-names master ignore;`
  dans `named.conf.options`
- Éviter les commentaires multi-lignes dans le fichier de zone (casse la syntaxe)
- Incrémenter le **Serial** à chaque modification de la zone

Toujours valider avant de recharger :
```bash
named-checkconf
named-checkzone starfleet.lan /etc/bind/db.starfleet.lan
systemctl reload bind9
```

### 1.7 Nginx (dernière version officielle)

Ajouter le dépôt officiel `nginx.org` (clé GPG + `/etc/apt/sources.list.d/nginx.list`).

⚠️ **Piège connu** : le Nginx du dépôt officiel tourne sous l'utilisateur **`nginx`**
(pas `www-data` comme la version Debian) — impacte toutes les configs de permissions
en aval (sockets PHP-FPM notamment).

```bash
nginx -t   # toujours valider avant de recharger
systemctl reload nginx
```

### 1.8 PHP 7.4 + 8.4 (cohabitation via PHP-FPM)

Ajouter le dépôt Sury (`packages.sury.org/php`). Deux services séparés actifs :
`php7.4-fpm` et `php8.4-fpm`, chacun avec son socket dans `/run/php/`.

⚠️ **Piège connu** : les sockets appartiennent à `www-data` par défaut →
502 Bad Gateway car Nginx tourne en `nginx`. Corriger dans chaque pool
(`/etc/php/X.Y/fpm/pool.d/www.conf`) :
```
listen.owner = nginx
listen.group = nginx
listen.mode = 0660
```

### 1.9 MariaDB (dernière version officielle)

Ajouter le dépôt officiel via `mariadb_repo_setup`, puis sécuriser l'installation :
```bash
mariadb-secure-installation
```
(mot de passe root, comptes anonymes supprimés, root distant désactivé, base test supprimée)

### 1.10 Certificat SSL

Créer un certificat auto-signé **wildcard** `*.starfleet.lan`, réutilisé pour
Nginx et FTP :
```
/etc/ssl/starfleet/starfleet.crt
/etc/ssl/starfleet/starfleet.key
```

### 1.11 Les 4 sites Nginx (HTTPS)

Dans `/etc/nginx/conf.d/`, un fichier par site, tous en HTTPS avec redirection 80→443 :

| Site | Dossier | PHP |
|------|---------|-----|
| `www8.starfleet.lan` | `/var/www/www8` | 8.4 |
| `www7.starfleet.lan` | `/var/www/www7` | 7.4 |
| `php.starfleet.lan` | `/var/www/phpmyadmin` | 8.4 (phpMyAdmin) |
| `_2admin.starfleet.lan` | `/var/www/_2admin` | 8.4 |

Pour phpMyAdmin, générer le secret dans `config.inc.php` :
```bash
openssl rand -base64 32
```

### 1.12 FTP (vsftpd)

SSL/TLS obligatoire (`force_local_data_ssl`, `force_local_logins_ssl`), certificat
wildcard réutilisé, chrooté sur `/var/www` via un utilisateur dédié `ftpuser`
(shell `/usr/sbin/nologin`).

⚠️ **Piège connu** : `/usr/sbin/nologin` doit être ajouté à `/etc/shells`, sinon
PAM refuse l'authentification FTP (`530 Login incorrect`) même avec le bon mot de passe.

### 1.13 LDAP (OpenLDAP)

Domaine `dc=starfleet,dc=lan`, structure `ou=people`, `ou=groups`. Utilisateur
de test : `uid=kirk,ou=people,dc=starfleet,dc=lan`.

Page `www8.starfleet.lan/ldap_login.php` : formulaire d'authentification via
`ldap_bind()`.

⚠️ **Piège connu** : le mot de passe **hashé** (`{SSHA}...`) stocké dans LDAP
n'est jamais celui à retaper pour se connecter — c'est le mot de passe en clair
d'origine.

---

## Partie 2 — VM Cliente

### 2.1 Création de la VM

- **Custom (advanced)**
- RAM : **2 Go**, 2 vCPU, disque **16 Go**
- Réseau : **Host-only**, sur le **même vmnet** que la carte LAN du serveur
- ISO **Debian 13.x** (DVD complet recommandé pour installer l'environnement
  de bureau sans dépendre d'internet)

### 2.2 Installation

- Installateur graphique
- Partitionnement assisté, disque entier, une seule partition
- Sélection des logiciels (tasksel) : **environnement de bureau Debian** (GNOME) +
  **utilitaires usuels du système**
- GRUB installé sur le disque principal

### 2.3 Vérifications post-installation

Depuis un terminal GNOME :

```bash
ip a
```
Doit montrer une IP dans la plage `192.168.100.10-100` (attribuée par le DHCP
du serveur).

```bash
ping starfleet.lan
```
Doit résoudre vers `192.168.100.1`.

### 2.4 Tests fonctionnels

Depuis Firefox (préinstallé avec GNOME) :

1. `https://www8.starfleet.lan` → phpinfo() PHP 8.4 (accepter le certificat
   auto-signé)
2. `https://www7.starfleet.lan` → phpinfo() PHP 7.4
3. `https://php.starfleet.lan` → page de connexion phpMyAdmin
4. `https://_2admin.starfleet.lan` → page d'administration (hostname, date, PHP)

Test FTP via le gestionnaire de fichiers GNOME (Nautilus) :
```
ftps://192.168.100.1
```
Utilisateur `ftpuser` — le chroot doit limiter la navigation aux 4 dossiers
des sites web.

Test LDAP via `https://www8.starfleet.lan/ldap_login.php` avec l'utilisateur
`kirk` (mot de passe en clair).

> **Bug connu** : sur cette VM, le champ mot de passe du formulaire LDAP ne
> s'affiche pas visuellement dans Firefox (rendu graphique), bien que le
> code HTML soit correct et le champ fonctionnel. Validation alternative :
> ```bash
> curl -k -X POST -d "uid=kirk&password=MOTDEPASSE" https://www8.starfleet.lan/ldap_login.php
> ```
> Un « Connexion reussie » ou « Echec de connexion » confirme le bon
> fonctionnement de l'authentification, indépendamment de l'affichage.

---

## Pièges génériques (utiles pour toute la suite)

- Toujours faire `nginx -t` / `named-checkconf` / `dhcpd -t -cf ...` **avant**
  de recharger un service
- Différencier une commande à taper dans **nano** (édition de fichier) d'une
  commande à taper dans le **terminal** (exécution)
- Sans souris dans la VM serveur : attention aux fautes de frappe
