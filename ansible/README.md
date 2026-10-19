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

## Ajouter un site

Créer le groupe `site_c` dans `inventory/hosts.yml` et `inventory/group_vars/site_c.yml`.
Le rôle ne change pas.
