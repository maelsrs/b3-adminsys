# Oral LogiNord : quoi dire, quoi montrer

Le client n'est pas informaticien (« expliquez-moi simplement »). Règle : **son problème → ce
qu'on a fait, en une phrase simple → la preuve à l'écran**. Le détail technique vient
seulement si le jury creuse (voir le README, sections « Points à expliquer à l'oral »).

## 1. Le pitch (1 minute)

> LogiNord, c'est deux sites, Lille et Dunkerque, reliés par une liaison qui coupe tous les
> mois. On a construit une infrastructure où **Dunkerque continue de travailler seul** quand la
> liaison tombe, où **chacun a un seul identifiant**, où **les salaires ne sortent pas des RH**,
> et où **tout est écrit sous forme de code** : si votre informaticien part, un autre relance
> une commande et reconstruit tout à l'identique. Le tout avec des logiciels libres et des
> machines virtuelles, sans licence en plus de Windows Server.

## 2. Les problèmes du client, nos réponses

| Le client a dit | Ce qu'on a fait | La preuve |
|---|---|---|
| « La liaison avec Dunkerque coupe tous les mois » | Un contrôleur de domaine **dans chaque site** (win02 Lille, win01 Dunkerque), DNS et DHCP locaux ; TrackR et sa base **à Dunkerque** | Démo couper le lien (§4) ; `nltest /dsgetsite` |
| « TrackR ne doit jamais bloquer les camions » | TrackR = service systemd qui **redémarre seul** ; base sur **RAID 1** | `kill -9` → relancé en 5 s (`verif_trackr.yml`) ; RAID dégradé, la base reste lisible |
| « Ne plus jamais ressaisir des jours de travail » | Sauvegarde **toutes les heures** (perte max : 1 h), **testée** par restauration | `pg_restore` dans `trackr_test`, même nombre de lignes |
| « Le serveur compta est tombé l'hiver dernier » | Partages sur deux disques en miroir (Storage Spaces) ; clichés VSS à 7 h et 12 h | `Get-VirtualDisk` : Mirror Healthy ; fichier supprimé puis restauré |
| « Le chef parti se connectait encore » | **Un seul annuaire** (AD) pour Windows et Linux : un compte désactivé l'est partout. Réplication **immédiate** entre sites (notification) | Désactiver un compte à Lille : refusé à Dunkerque quelques secondes après (à répéter avant l’oral) |
| « Les salaires ont fuité » | Partage RH : **seul le groupe RH**, même pas les admins IT ; les autres **ne voient même pas** le dossier (ABE) | alice (compta) ne voit que `Compta`, `\\srv01\RH` : accès refusé |
| « Les serveurs se remplissent de n'importe quoi » | **Quotas stricts** : 5 Go par service (Windows), 200 Mo pour le dépôt des devs (Linux) | Copie de 5,1 Go refusée ; `dd` coupé à 200 Mo |
| « Un seul identifiant par personne » | Comptes AD pour tout le monde, y compris pour se connecter aux Debian (Kerberos/SSSD) | test2 entre en SSH sur deb01 avec son compte Windows |
| « Le presta ne voit pas tout » | Prestataires dans un **annuaire séparé** (OpenLDAP), jamais dans l'AD | `ldapsearch` : jdupont n'existe que dans ldap01 |
| « Intervenir de chez soi la nuit sans ouvrir la porte aux pirates » | **VPN** obligatoire ; SSH par clé uniquement, root refusé, groupes autorisés seulement | `ssh root@…` refusé, `ssh` sans clé refusé |
| « Savoir qui fait quoi » | Comptes **nominatifs**, compte admin séparé (`adm-test2`) avec mot de passe renforcé (16 car., PSO) ; historique Git de chaque changement | `git log` ; `Get-ADUserResultantPasswordPolicy adm-test2` |
| « Si mon informaticien part, un autre doit tout reconstruire » / « tout doit être écrit » | **Tout est en Ansible** + README ; mots de passe chiffrés (vault) | `ansible-playbook site.yml` → `changed=0` au 2e passage |
| « On pense ouvrir un troisième site » | Les sites sont une **liste** dans le code : OU, sous-réseaux, DHCP, liens AD se génèrent | Montrer `ad_sites` dans l'inventaire |
| « Budget serré, aucun arrêt, des résultats vite » | Logiciels libres, clones liés Proxmox (un disque ne stocke que ses différences), déploiement par morceaux (`--tags`, `-l`) sans arrêter le reste | — |

**À venir (le dire franchement)** : arrivées/départs validés par les RH (MC13), preuve des mises
à jour pour l'assureur (MC14), alerte quand TrackR tombe (MC15).

## 3. Les images pour un non-informaticien

| Notion | En une image |
|---|---|
| Ansible | Une **recette** écrite : on la relance, on obtient le même plat |
| Idempotence (`changed=0`) | Relancer la recette ne rajoute pas le sel une deuxième fois |
| Contrôleur de domaine | Le **registre du personnel** + le guichet qui vérifie les badges, un par site |
| Kerberos | Un **bracelet de festival** : on montre ses papiers une fois à l'entrée, puis le bracelet ouvre les stands. Le mot de passe ne circule jamais |
| Notification entre sites | Le DC de Lille **téléphone** à Dunkerque à chaque changement, au lieu d'attendre la tournée du facteur (toutes les 15 min) |
| AGDLP | Le badge du service ouvre les portes du service ; on change une personne de service, pas les serrures |
| ABE | On ne voit même pas les portes qu'on ne peut pas ouvrir |
| RAID 1 / miroir | Deux carnets écrits en même temps : on en perd un, l'autre suffit. **Mais** une page arrachée l'est dans les deux : ce n'est pas une sauvegarde |
| Clichés VSS | Une **photo** du classeur à 7 h et à midi |
| Sauvegarde horaire | Le client accepte de perdre au plus une heure de saisie |
| systemd `Restart=` | Un **gardien** qui relance la machine si elle cale |
| Timer | Un réveil, avec carnet de bord (`journalctl`) |
| Quota | Une étagère par service, de taille fixe |
| VPN | Un **tunnel privé** : de l'extérieur, on ne voit pas ce qui y passe |
| Vault | Un coffre-fort dans le dépôt : le code est lisible, les mots de passe non |

## 4. Les démos (ordre conseillé)

Avant l'oral : VPN connecté, toutes les VM allumées, puis
`ansible all -m ping -l linux` et `ansible all -m ansible.windows.win_ping -l windows -e @vault/secrets.yml`.

1. **Tout est code** (1 min) : `ansible-playbook site.yml -l deb01` → `changed=0`.
2. **Un seul identifiant** (2 min) : `ansible-playbook playbooks/verif_linux_ad.yml` : test2 entre
   avec sudo, bob entre sans sudo, alice refusée.
3. **Salaires protégés** (2 min) : en alice, `\\srv01\Partages` ne montre que Compta ; RH refusé.
4. **TrackR increvable** (2 min) : `ansible-playbook playbooks/verif_trackr.yml` : kill -9,
   PID qui change, restauration, prochaine sauvegarde.
5. **Panne de disque** (3 min, si le temps) : `mdadm --fail` sur db01, `[U_]`, la base répond,
   `--add`, reconstruction.
6. **Couper le lien** (la démo de la soutenance selon le MC05) : README « Démo : couper le lien ».
   ⚠️ Pas encore répétée depuis le passage par le VPN (à refaire avec un VPN par site).

## 5. Questions probables du jury

- **Pourquoi `loginord.internal` et pas `.local` ?** `.local` est réservé au mDNS (Bonjour,
  Avahi) : conflits de résolution sur Linux et Mac. `.internal` est réservé pour l'usage privé.
- **Pourquoi deux annuaires (AD + OpenLDAP) alors que le client veut un seul identifiant ?**
  Un seul identifiant **par salarié**. Les prestataires n'ont pas à entrer dans l'AD : un compte
  AD donne par défaut accès à beaucoup de choses (lecture de l'annuaire, partages « Utilisateurs
  du domaine »). Annuaire séparé = surface réduite, contrat de 3 mois facile à couper.
- **Le RAID suffit-il ?** Non : il protège du matériel, pas d'une suppression ni d'un
  rançongiciel. D'où la sauvegarde horaire testée (et la copie sur l'autre site, à faire).
- **Si win02 (Lille) tombe ?** Dunkerque authentifie seul. Les rôles FSMO sont tous sur win02 :
  pas de nouveau DC ni de changement de schéma tant qu'il est absent ; s'il est perdu, on
  **saisit** les rôles sur win01 (`Move-ADDirectoryServerOperationMasterRole -Force`).
- **Pourquoi l'heure est-elle critique ?** Kerberos refuse plus de 5 min d'écart. La racine
  (PDC) se cale sur des serveurs NTP publics, les autres DC sur la hiérarchie (`NT5DS`), les
  Debian sur les DC (chrony).
- **Un salarié part, que se passe-t-il sur Linux hors ligne ?** SSSD garde un cache : utile lien
  coupé, mais un compte parti peut encore entrer hors ligne. `sss_cache -E` fait partie de la
  procédure de départ (MC13).
- **Où sont les mots de passe ?** Chiffrés dans `vault/secrets.yml` (AES-256), jamais en clair
  dans le dépôt ni dans les journaux (`no_log`). Le compte TrackR n'en a pas : authentification
  *peer* (le compte Linux = le compte PostgreSQL).
- **Les admins IT ne peuvent pas ouvrir RH : et en cas de problème ?** Ils restent
  **propriétaires** du dossier : ils peuvent reprendre la main, mais par un geste volontaire,
  visible dans les journaux, jamais par défaut.
- **Pourquoi TrackR à Dunkerque et pas au siège ?** Les camions sont à Dunkerque : lien coupé,
  l'appli et sa base sont du même côté que ses utilisateurs.
- **Pourquoi des timers plutôt que cron ?** Journaux dans `journalctl`, prochaine exécution
  visible (`list-timers`), rattrapage d'une exécution manquée (`Persistent=true`).
- **Que veut dire le score 1.0 de `systemd-analyze security` ?** 0 = parfaitement isolé,
  10 = aucune protection. TrackR ne peut écrire que dans `/var/lib/trackr`, n'a aucun
  privilège, ne parle qu'au LAN.

## 6. Limites connues (les annoncer avant qu'on les trouve)

- Tous les rôles FSMO sur win02 (acceptable à 2 DC, à revoir avec un 3e site).
- Sauvegardes de TrackR sur la même machine : la copie vers Lille est le bonus MC11, pas fait.
- Certificat OpenLDAP auto-signé, et lecture anonyme de l'annuaire prestataires (hors mots de passe).
- rtr-a / rtr-b : l'accès SSH d'Ansible est à réparer à la console (`adduser ansible ssh-admins`).
- TrackR est une maquette : le vrai programme remplacera `/opt/trackr/trackr`, l'unité ne change pas.
