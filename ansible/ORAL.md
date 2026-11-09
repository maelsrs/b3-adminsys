# Oral LogiNord : 10 minutes

Le client n'est pas informaticien. Règle de chaque bloc : **son problème → ce qu'on a fait, en
une phrase → la preuve à l'écran**. Le détail technique seulement si le jury creuse (README).

Tout ce qui suit a été testé le 9/11 depuis le portable, VPN connecté.

## 1. Les accès

Mots de passe : `ansible-vault view vault/secrets.yml`, **dans un terminal non projeté**, avant
l'oral. Jamais à l'écran.

| Machine | IP | Pour entrer | Compte | Mot de passe (vault) |
|---|---|---|---|---|
| VPN | → 10.1.0.0/16 | `cd ~/Downloads && sudo openvpn mael.ovpn` | | |
| deb01 (Debian, Lille) | 10.1.10.20 | `ssh ansible@10.1.10.20` | ansible | clé SSH, sudo sans mot de passe |
| ldap01 (prestataires) | 10.1.10.21 | `ssh ansible@10.1.10.21` | ansible | clé SSH |
| db01 (TrackR, Dunkerque) | 10.1.20.21 | `ssh ansible@10.1.20.21` | ansible | clé SSH |
| **TrackR** | 10.1.20.21 | **http://10.1.20.21:8080** (Firefox) | paul.durand | `vault_pwd_init` |
| win02 (DC Lille) | 10.1.10.10 | `ssh Administrator@10.1.10.10` puis `powershell` | Administrator | `vault_win_password` |
| win01 (DC Dunkerque) | 10.1.20.30 | `ssh Administrator@10.1.20.30` puis `powershell` | Administrator | `vault_win_password` |
| srv01 (fichiers Lille) | 10.1.10.30 | `ssh Administrateur@10.1.10.30` puis `powershell` | Administrateur | `vault_win_password` |
| Partages Windows | 10.1.10.30 | `smbclient //10.1.10.30/Partages -U 'LOGINORD\alice.martin' -c ls` | un salarié | `vault_pwd_init` |
| mon01 (supervision) | 10.1.10.22 | `ssh ansible@10.1.10.22` ; alertes : **http://10.1.10.22:9090/classic/alerts** (Firefox) | ansible | clé SSH |
| OpenLDAP | 10.1.10.21 | `ldapsearch -x -ZZ -H ldap://ldap01.loginord.internal …` | cn=admin | `vault_ldap` |

**Montrer Windows** : les serveurs sont en Server Core (pas de bureau). On y entre en **SSH**
depuis le portable (mot de passe), on tape `powershell`, et on lance les commandes du bloc. Les
partages se montrent avec **`smbclient`** depuis le portable : c'est exactement ce qu'un salarié
verrait, mais sans avoir besoin d'un poste Windows.

**Comptes de démo** (tous en `vault_pwd_init`) :

| Compte | Qui | Ce qu'il doit se passer |
|---|---|---|
| paul.durand | quais, Dunkerque | entre dans TrackR |
| alice.martin | compta, Lille | voit seulement `Compta` ; RH refusé ; TrackR refusé |
| test2.test2 | informaticien | voit seulement `Adminsys` ; **RH refusé aussi** ; sudo sur deb01 |
| chloe.petit | partie le 9/11 à 16:00 | `NT_STATUS_ACCOUNT_DISABLED` partout |

## 2. La veille

- [ ] VPN : `cd ~/Downloads && sudo openvpn mael.ovpn`, puis `ping 10.1.20.21`
- [ ] Les empreintes SSH des Windows ont changé depuis la réinstallation (« Host key verification
  failed ») : `ssh-keygen -R 10.1.10.10 && ssh-keygen -R 10.1.20.30 && ssh-keygen -R 10.1.10.30`,
  puis une connexion à chaque pour accepter la nouvelle empreinte
- [ ] `ansible all -m ping -l linux` et `ansible windows -m ansible.windows.win_ping -e @vault/secrets.yml`
  (routeurs et openvpn : en échec, connu)
- [ ] `ansible-playbook playbooks/verif_trackr.yml` : tout vert
- [ ] RAID de db01 en `[UU]` : `ssh ansible@10.1.20.21 cat /proc/mdstat`
- [ ] `ansible-playbook playbooks/verif_supervision.yml` : toutes les cibles `up`, **aucune alerte en cours**
- [ ] Répéter une fois en chronométrant
- [ ] **Si l'oral tombe un dimanche** : ne pas lancer `campagne.yml` (il mettrait vraiment à jour)

## 3. L'écran, 10 minutes avant

Terminal en grosse police (Ctrl + +), un onglet par rôle :

| Onglet | Commande d'ouverture |
|---|---|
| T1 portable | `cd ~/Documents/b3-adminsys/ansible` |
| T2 db01 | `ssh ansible@10.1.20.21` |
| T3 win01 | `ssh Administrator@10.1.20.30`, puis `powershell` |
| T4 Ansible | `cd ~/Documents/b3-adminsys/ansible` (lancé au début, regardé à la fin) |
| T5 alertes | `ssh ansible@10.1.10.22 sudo journalctl -u alertes -f -o short` (les alertes arrivent ici) |

Firefox, dans l'ordre :

1. Le dépôt GitHub, le README (le schéma s'affiche)
2. `inventory/group_vars/all/personnel.yml` sur GitHub
3. http://10.1.20.21:8080 (page de connexion TrackR)
4. `ansible/rapports/maj-2026-11-09.html` (glisser le fichier dans Firefox)
5. http://10.1.10.22:9090/classic/alerts (les règles d'alerte, vertes)

## 4. Le déroulé

Les temps sont des repères. Si on est en retard, sauter ce qui est marqué *(si le temps)*.

### 0:00 · Le pitch (1 min) · Firefox onglet 1, le schéma

À lancer **avant de parler**, dans T4 : `ansible-playbook site.yml -l deb01` (on regarde le
résultat à la fin).

> LogiNord, c'est deux sites, Lille et Dunkerque, reliés par une liaison qui coupe tous les
> mois. On a construit une infrastructure où **Dunkerque continue de travailler seul**, où
> **chacun a un seul identifiant**, où **les salaires ne sortent pas des RH**, et où **tout est
> écrit** : si votre informaticien part, un autre relance une commande et reconstruit tout.
> Avec des logiciels libres, sans licence en plus de Windows Server.

Sur le schéma : à gauche Lille (siège, DC racine, serveur de fichiers), à droite Dunkerque (son
propre DC, TrackR et sa base), entre les deux un tunnel chiffré.

### 1:00 · « Quelqu'un part : accès coupé le soir même » (2 min)

> Ce ne sont pas les informaticiens qui décident, ce sont les RH. Ils ont un fichier, une ligne
> par personne. Chloé quitte l'entreprise : les RH ont écrit sa date et son heure de départ.

Firefox onglet 2 : la ligne `chloe.petit … depart: 2026-11-09T16:00`, puis *History* : on voit qui
l'a demandé et quand.

> À 16 h, son compte a été désactivé, partout. Jamais supprimé : la compta récupère ses
> fichiers, et la trace reste.

T3 (win01, **le DC de Dunkerque** : le changement fait à Lille est déjà arrivé) :

```powershell
Get-ADUser chloe.petit | ft Name,Enabled,DistinguishedName
```

→ `False`, rangée dans `OU=Départs`.

T1 : elle essaie d'ouvrir les fichiers (mot de passe : `vault_pwd_init`)

```sh
smbclient //10.1.10.30/Partages -U 'LOGINORD\chloe.petit' -c ls
```

→ `NT_STATUS_ACCOUNT_DISABLED`.

*(si le temps)* Même chose côté Linux, même hors ligne :
`ssh ansible@10.1.10.20 sudo sssctl user-checks chloe.petit | grep pam_acct` → `Permission denied`.

### 3:00 · « Les salaires ont fuité » (1 min 30)

> Le dossier RH, seuls les RH l'ouvrent. Même pas nous, les informaticiens. Et les autres ne le
> voient même pas.

T1 :

```sh
smbclient //10.1.10.30/Partages -U 'LOGINORD\alice.martin' -c ls   # Alice (compta) : seulement Compta
smbclient //10.1.10.30/RH -U 'LOGINORD\alice.martin' -c ls         # NT_STATUS_ACCESS_DENIED
smbclient //10.1.10.30/RH -U 'LOGINORD\test2.test2' -c ls          # l'informaticien : refusé aussi
```

> Et le serveur compta tombé l'hiver dernier : ses données sont sur deux disques en miroir.

*(si le temps)* T1 : `ansible srv01 -e @vault/secrets.yml -m ansible.windows.win_shell -a 'Get-VirtualDisk | ft FriendlyName,ResiliencySettingName,HealthStatus'`
→ `vData Mirror Healthy` (10 s).

### 4:30 · TrackR : « ne doit jamais bloquer les camions » (3 min) · le moment fort

**Un seul identifiant.** Firefox onglet 3 :

1. `alice.martin` → refusée (elle est à la compta, pas aux quais).
2. `paul.durand` (son mot de passe Windows) → il entre. Créer une expédition `ORAL-001`.

> Paul n'a pas de mot de passe TrackR : c'est celui de Windows. Le jour où il part, il ne rentre
> plus dans TrackR non plus, sans rien toucher.

**On casse TrackR.** T2 :

```sh
sudo kill -9 $(systemctl show -p MainPID --value trackr)    # on tue l'appli, brutalement
systemctl status trackr --no-pager | head -3                 # « activating (auto-restart) »
```

Recharger Firefox : erreur. Attendre 5 secondes, recharger : **TrackR est revenu**, `ORAL-001`
est toujours là.

> Un gardien relance l'appli si elle cale, en 5 secondes, sans personne.

```sh
sudo journalctl -u trackr -n 6 --no-pager # l'arrêt, le redémarrage, la connexion de Paul
```

**On casse un disque.** T2 :

```sh
cat /proc/mdstat                                          # [UU] : deux disques
D=$(readlink -f /dev/disk/by-id/scsi-0QEMU_QEMU_HARDDISK_drive-scsi2)
sudo mdadm /dev/md0 --fail $D && sudo mdadm /dev/md0 --remove $D
cat /proc/mdstat                                          # [U_] : un disque mort
```

Firefox : créer `ORAL-002` → **ça marche**, la base tourne sur un seul disque. **On laisse le
disque retiré** : on le remet au bloc « être prévenu ».

> Et si quelqu'un supprime des données par erreur ? Le miroir supprime des deux côtés. D'où une
> sauvegarde toutes les heures, et on vérifie qu'elle se restaure.

```sh
sudo ls -lt /var/backups/pg | head -3                    # une sauvegarde par heure
systemctl list-timers pg-backup.timer --no-pager         # la prochaine
```

### 7:30 · « La liaison coupe tous les mois » (30 s) · T3, win01

> Dunkerque a son propre contrôleur de domaine : le registre du personnel et le guichet qui
> vérifie les badges. Liaison coupée, les gens de Dunkerque se connectent quand même, et TrackR
> est à Dunkerque, à côté des camions.

```powershell
nltest /dsgetsite            # Dunkerque
Get-ADReplicationPartnerMetadata -Target win01 | ft Partner,LastReplicationSuccess,LastReplicationResult
# dernière synchro avec Lille il y a quelques minutes, résultat 0 = aucune erreur
```

> Quand on change quelque chose à Lille, Lille téléphone à Dunkerque tout de suite, au lieu
> d'attendre la tournée du facteur.

On ne coupe pas le lien en direct : notre VPN arrive à Lille, couper le tunnel nous couperait
aussi de Dunkerque. Si le jury insiste : README « Démo : couper le lien », à la console Proxmox.

### 8:00 · « Pas de vision quand l'application tombe » (1 min) · T5, puis T2

> Le disque qu'on a retiré tout à l'heure : personne n'a rien remarqué, la base tourne. C'est
> justement le danger, on ne le découvre qu'à la panne du second. Alors une machine surveille
> tout, et nous prévient.

T5 : la ligne arrivée toute seule, ~2 min après le retrait :
`ALERTE RaidDegrade : db01 : le RAID md0 a perdu un disque`. Firefox onglet 5 : la règle en rouge.

T2 : on remet le disque, l'alerte se résout d'elle-même (`RÉSOLU RaidDegrade`, ~1 min après la
reconstruction).

```sh
sudo mdadm /dev/md0 --add $D && cat /proc/mdstat         # reconstruction en cours
```

> Elle surveille aussi la liaison avec Dunkerque, les disques qui se remplissent, TrackR vu
> comme un utilisateur, et les services des deux contrôleurs. Les journaux de toutes les Debian
> arrivent au même endroit, gardés 3 mois. Pour le kill de TrackR, pas d'alerte : il est revenu
> en 5 secondes, on ne réveille personne pour ça.

### 9:00 · « L'assureur veut la preuve » et « tout doit être écrit » (45 s)

Firefox onglet 4 : le rapport. Une ligne par machine, en français, daté.

T1 :

```sh
ansible-playbook playbooks/campagne.yml   # refusé : « Nous sommes lundi … : les camions roulent »
```

> Les mises à jour lourdes se font le dimanche, dépôt fermé, une machine à la fois, jamais les
> deux contrôleurs ensemble. Le reste de la semaine, la commande refuse.

T4 : le récapitulatif lancé au début → `changed=0`.

> On a relancé toute la recette sur une machine : rien n'a changé, tout est déjà conforme. Si
> votre informaticien part, le suivant relance la même commande.

### 9:45 · Conclusion (15 s)

> Ce qui reste : une copie des sauvegardes sur l'autre site, et les alertes sur le téléphone
> des informaticiens (Discord) en plus du journal.

## 5. Si ça casse pendant l'oral

| Panne | Réaction |
|---|---|
| Le VPN tombe | `cd ~/Downloads && sudo openvpn mael.ovpn` dans un onglet à part ; pendant ce temps, parler sur le schéma |
| TrackR ne revient pas après le kill | `sudo systemctl start trackr` ; puis `sudo journalctl -u trackr -n 20` |
| smbclient muet ou lent | ajouter `-m SMB3` ; sinon montrer le tableau de tests du README (MC10) |
| SSH Windows refusé (empreinte) | `ssh-keygen -R <ip>` puis reconnecter |
| Le RAID ne reconstruit pas | `sudo mdadm /dev/md0 --add $D` de nouveau |
| L'alerte RAID n'est pas encore là à 8:00 | Firefox onglet 5 : la règle est déjà « pending » (orange) ; parler 30 s des journaux (`ssh ansible@10.1.10.22 ls /var/log/central`), puis revenir à T5 |
| `site.yml` pas fini à 8:30 | ne pas attendre : le montrer en fin de questions |
| Plus de temps du tout | sauter les *(si le temps)* et le bloc 7:30 (le garder pour les questions) |

## 6. Les images pour un non-informaticien

| Notion | En une image |
|---|---|
| Ansible | Une **recette** écrite : on la relance, on obtient le même plat |
| `changed=0` | Relancer la recette ne rajoute pas le sel une deuxième fois |
| Contrôleur de domaine | Le **registre du personnel** + le guichet qui vérifie les badges, un par site |
| Kerberos | Un **bracelet de festival** : papiers une fois à l'entrée, puis le bracelet ouvre les stands |
| Réplication avec notification | Lille **téléphone** à Dunkerque au lieu d'attendre le facteur |
| AGDLP | Le badge du service ouvre les portes du service ; on change une personne de service, pas les serrures |
| ABE | On ne voit même pas les portes qu'on ne peut pas ouvrir |
| RAID 1 | Deux carnets écrits en même temps. **Mais** une page arrachée l'est dans les deux |
| Clichés VSS | Une **photo** du classeur à 7 h et à midi |
| systemd `Restart=` | Un **gardien** qui relance la machine si elle cale |
| Quota | Une étagère par service, de taille fixe |
| Supervision | Un **gardien de nuit** qui fait sa ronde toutes les 30 s et appelle si une porte est ouverte |
| Durée d'une alerte (`for`) | On n'appelle pas pour une porte qui claque : seulement si elle reste ouverte 5 min |
| VPN | Un **tunnel privé** : de l'extérieur, on ne voit pas ce qui y passe |
| Vault | Un coffre-fort dans le dépôt : le code est lisible, les mots de passe non |

## 7. Questions probables du jury

- **Pourquoi `loginord.internal` et pas `.local` ?** `.local` est réservé au mDNS (Bonjour,
  Avahi) : conflits sur Linux et Mac. `.internal` est réservé à l'usage privé.
- **Deux annuaires (AD + OpenLDAP), alors qu'on veut un seul identifiant ?** Un seul identifiant
  **par salarié**. Les prestataires n'entrent pas dans l'AD, où un compte voit beaucoup par
  défaut. Annuaire séparé : surface réduite, mission de 3 mois facile à couper (`expiration`).
- **Le RAID suffit-il ?** Non : il protège du matériel, pas d'une suppression ni d'un rançongiciel.
  D'où la sauvegarde horaire testée.
- **Si win02 (Lille) tombe ?** Dunkerque authentifie seul. Les rôles FSMO sont sur win02 : s'il
  est perdu, on les **saisit** sur win01 (`Move-ADDirectoryServerOperationMasterRole -Force`).
- **Pourquoi l'heure est-elle critique ?** Kerberos refuse plus de 5 min d'écart. Le PDC suit le
  NTP public, les autres DC la hiérarchie (`NT5DS`), les Debian les DC (chrony).
- **Un salarié parti, sur un Linux hors ligne ?** Le cache SSSD le laisserait entrer : les partis
  sont refusés explicitement (`simple_deny_users`), sessions fermées, cache vidé.
- **Où sont les mots de passe ?** Chiffrés dans `vault/secrets.yml` (AES-256), jamais dans les
  journaux (`no_log`). TrackR n'a pas de mot de passe de base : authentification *peer*.
- **Les admins ne peuvent pas ouvrir RH : et en cas de problème ?** Ils restent
  **propriétaires** : ils peuvent reprendre la main, par un geste volontaire et journalisé.
- **Qui valide un départ ?** Les RH, par pull request : CODEOWNERS rend leur validation
  obligatoire sur `personnel.yml`. Git garde qui a demandé, qui a validé, quand.
- **Un troisième site ?** Les sites sont une **liste** (`ad_sites`) : OU, sous-réseaux, DHCP et
  liens AD se génèrent. Un site = une entrée de plus.
- **Pourquoi des timers plutôt que cron ?** Journaux dans `journalctl`, prochaine exécution
  visible, rattrapage d'une exécution manquée (`Persistent=true`).
- **Le score 1.0 de `systemd-analyze security` ?** 0 = isolé au maximum, 10 = rien. TrackR n'écrit
  que dans `/var/lib/trackr`, sans privilège, ne parle qu'au réseau interne.
- **Pourquoi Prometheus et pas Zabbix ?** Paquets Debian, pas de base de données, tient dans 1 Go
  de RAM (mon01). Zabbix est plus complet mais plus gourmand. Les deux se défendent.
- **Et si mon01 tombe ?** Plus personne n'est prévenu de rien : c'est la limite. Une sonde externe
  sur mon01 (ou un second Prometheus à Dunkerque) la lèverait.
- **Les journaux pendant une coupure de liaison ?** Chaque Debian les garde en file d'attente sur
  disque et les envoie au retour : rien n'est perdu.
- **Intervenir la nuit de chez soi ?** VPN obligatoire, SSH par clé seulement, root refusé.

## 8. Limites (les annoncer avant qu'on les trouve)

- Alertes dans le journal de mon01 seulement (Discord prévu) ; mon01 n'est surveillé par personne.
- Journaux Windows pas centralisés (seulement les Debian).
- Sauvegardes de TrackR sur la même machine : la copie vers Lille n'est pas faite.
- Tous les rôles FSMO sur win02 (acceptable à 2 DC).
- Certificats auto-signés (OpenLDAP, LDAPS des DC) ; lecture anonyme de l'annuaire prestataires.
- rtr-a / rtr-b et openvpn : Ansible n'y entre plus, à réparer à la console.
