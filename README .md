# 🖖 Holodeck — Machines Virtuelles Web pour les Ingénieurs de Starfleet

Projet réalisé dans le cadre du cursus **Bachelor IT Cybersécurité** — La Plateforme (Cannes).

## Contexte

À bord de l'USS Enterprise-D, la Fédération souhaite équiper ses ingénieurs du Holodeck
de machines virtuelles pour développer leurs sites web stellaires.

Ce projet met en place une infrastructure complète sur 2 VM Debian :

- Une **VM Serveur** : DHCP, DNS, Web (Nginx + PHP 7/8), MariaDB, FTP (SSL/TLS), LDAP
- Une **VM Cliente** : Debian avec environnement graphique, pour tester l'ensemble
  des services depuis un navigateur

## Architecture réseau

```
                    ┌─────────────────────────────┐
                    │      VM Serveur (Debian)     │
                    │                              │
   Internet ────────┤ ens33 (WAN, DHCP)            │
                    │                              │
                    │ ens34 (LAN) 192.168.100.1/24 │
                    │  ├─ DHCP  (isc-dhcp-server)  │
                    │  ├─ DNS   (bind9)            │
                    │  ├─ Nginx (HTTPS, 4 sites)   │
                    │  ├─ PHP 7.4 + PHP 8.4 (FPM)  │
                    │  ├─ MariaDB                  │
                    │  ├─ vsftpd (FTPS, chrooté)   │
                    │  └─ slapd (LDAP)             │
                    └──────────────┬───────────────┘
                                   │
                          Réseau Host-only
                                   │
                    ┌──────────────┴───────────────┐
                    │      VM Cliente (Debian)     │
                    │      + GNOME + Firefox       │
                    │   IP DHCP : 192.168.100.10   │
                    └───────────────────────────────┘
```

**Domaine** : `starfleet.lan`

## Services et versions

| Service      | Version                        | Détail                                   |
|--------------|---------------------------------|-------------------------------------------|
| Debian       | 13.x (Trixie)                   | Serveur : sans GUI · Client : GNOME       |
| Nginx        | 1.30.4 (dépôt officiel nginx.org)| HTTPS, redirection 80→443                 |
| PHP          | 7.4 + 8.4 (dépôt Sury)           | Cohabitation via PHP-FPM (sockets séparés)|
| MariaDB      | 12.3.3 (dépôt officiel)          | Sécurisée via `mariadb-secure-installation`|
| FTP          | vsftpd                          | SSL/TLS obligatoire, chrooté sur `/var/www`|
| LDAP         | OpenLDAP (slapd)                | `dc=starfleet,dc=lan`                     |
| DHCP/DNS     | isc-dhcp-server / bind9          | Domaine `starfleet.lan`                   |
| Pare-feu     | nftables                         | Politique par défaut : drop, ports requis uniquement |

## Sous-domaines

| Sous-domaine              | Rôle                          |
|----------------------------|-------------------------------|
| `www8.starfleet.lan`      | Site web PHP 8.4               |
| `www7.starfleet.lan`      | Site web PHP 7.4                |
| `php.starfleet.lan`       | phpMyAdmin                      |
| `_2admin.starfleet.lan`   | Page d'administration de la VM  |

## Contraintes respectées

- ❌ Pas de compte sudo sur le serveur (travail en `root` via `su -`)
- ✅ Pare-feu nftables limité aux ports strictement nécessaires (22, 53, 67, 80, 443, 21 + plage FTP passive)
- ✅ Nginx, PHP et MariaDB en dernière version officielle (dépôts tiers, pas Debian)
- ✅ PHP 7.x et 8.x en cohabitation
- ✅ Certificat SSL wildcard auto-signé (`*.starfleet.lan`), réutilisé pour Nginx et FTP
- ✅ Authentification LDAP sur le site web

## Tests réalisés (validés de bout en bout depuis la VM Cliente)

- ✅ Attribution IP dynamique via DHCP (plage `192.168.100.10-100`)
- ✅ Résolution DNS de `starfleet.lan` et des sous-domaines
- ✅ Accès HTTPS aux 4 sites (certificat auto-signé accepté)
- ✅ Connexion FTPS avec chroot confirmé (utilisateur `ftpuser`)
- ✅ Authentification LDAP (bon mot de passe accepté, mauvais rejeté) — voir [docs/installation.md](docs/installation.md) pour le détail des tests

> **Note connue** : le formulaire de login LDAP (`ldap_login.php`) présente un bug d'affichage
> sous Firefox dans la VM Cliente (champ mot de passe présent dans le code mais non rendu
> visuellement, probablement lié au pilote graphique VMware SVGA). L'authentification a été
> validée fonctionnellement via `curl` en ligne de commande. Le code du formulaire est correct
> (vérifié à la source et via requête HTTP brute).

## Documentation complémentaire

- 📖 [Notice d'installation et d'utilisation](docs/installation.md)
- 💾 [Procédure d'export des VM](docs/export-vm.md)

## Compétences visées

- Administrer et sécuriser les infrastructures systèmes
- Concevoir une solution technique répondant à des besoins d'évolution de l'infrastructure
- Participer à l'élaboration et à la mise en œuvre de la politique de sécurité

## Identifiants de test (à ne pas utiliser en production)

| Service | Utilisateur | Mot de passe |
|---------|-------------|---------------|
| LDAP    | `kirk`      | `CHANGEME`    |
| FTP     | `ftpuser`   | `CHANGEME`    |
| MariaDB root | `root` | `CHANGEME`    |

---

*Projet réalisé par [ton nom] — La Plateforme, Cannes.*
