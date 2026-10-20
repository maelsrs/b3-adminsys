# loginord-infra

Construit une Debian (site A) et un Windows Server Core (site B) avec un rôle `baseline` :
nom d'hôte, fuseau horaire, NTP, compte admin.

## Prérequis sur le contrôleur

```sh
pip install pywinrm            # ou : pacman -S python-pywinrm
ansible-galaxy collection install -r requirements.yml -p collections
echo 'motdepasseduvault' > ~/.vault_pass && chmod 600 ~/.vault_pass
```

## Le seul geste manuel : rendre les VM joignables

**Debian (deb01)** — en root (`su -`, avec le tiret, sinon `/usr/sbin` n'est pas dans le PATH) :

```sh
apt install -y sudo openssh-server
useradd -m -s /bin/bash -G sudo ansible
passwd ansible
echo 'ansible ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/ansible
chmod 440 /etc/sudoers.d/ansible
```

et depuis le contrôleur : `ssh-copy-id ansible@<ip>`.

**Windows Server Core (win01)** — WinRM est actif par défaut ; `-q` accepte toutes les
modifications proposées, y compris `LocalAccountTokenFilterPolicy` (droits admin à distance
pour les comptes locaux autres que `Administrateur`) :

```powershell
winrm quickconfig -q
```

Adapter les IP dans `inventory/hosts.yml`.

## Secrets

`vault/secrets.yml` (chiffré) doit contenir :

```yaml
vault_win_password: "<mot de passe Administrateur Windows>"
vault_admin_password: "<mot de passe du compte admin Windows>"
vault_admin_password_hash: "<sortie de : openssl passwd -6>"
```

```sh
ansible-vault edit vault/secrets.yml
```

## Lancer

```sh
ansible all -m ping -l linux
ansible all -m ansible.windows.win_ping -l windows
ansible-playbook site.yml
ansible-playbook site.yml   # le 2e passage doit afficher changed=0
```

## Routeurs (MC02)

`rtr-a` et `rtr-b` (Alpine) : rôles `router` (IP, routage, nftables), `wireguard` (tunnel `wg0`)
et `frr` (OSPF). Ansible passe par `eth0`, que ces rôles ne touchent jamais.

| | rtr-a | rtr-b |
|---|---|---|
| eth1 (LAN) | 10.1.10.1/24 | 10.1.20.1/24 |
| eth2 (WAN) | 192.0.2.1/24 | 192.0.2.2/24 |
| wg0 | 10.255.0.1/30 | 10.255.0.2/30 |

- Réseau : `ifupdown-ng`, un fichier par interface dans `/etc/network/interfaces.d/`
  (`lan`, `wan`, `wg0`). C'est l'outil natif d'Alpine, sans systemd.
- WireGuard : `wg-quick` (via `use wireguard-quick`), `Table = off` et `AllowedIPs = 0.0.0.0/0` :
  aucune route statique, c'est OSPF qui les pose.
- MTU : les réseaux internes passent dans le vxlan Proxmox (1400), d'où un tunnel à 1320.
- Clés privées : `vault/wireguard.yml` (même mot de passe que `vault/secrets.yml`), clés
  publiques dans `inventory/host_vars/`.

```sh
ansible-playbook site.yml -l routers
ssh ansible@192.168.0.40 sudo vtysh -c '"show ip ospf neighbor"' -c '"show ip route ospf"'
```

## Ajouter un site

Créer le groupe `site_c` dans `inventory/hosts.yml` et `inventory/group_vars/site_c.yml`.
Le rôle ne change pas.
