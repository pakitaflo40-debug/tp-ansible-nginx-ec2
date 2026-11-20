TP — Administrer une instance EC2 avec Ansible (installation de Nginx)
Objectifs pédagogiques
Installer et configurer Ansible sur une machine Ubuntu (control node)
Se connecter en SSH à une instance EC2 AWS via un inventaire Ansible
Installer Nginx sur l'EC2 de trois façons différentes :
Playbook "brut" (sans module préfait, via commandes shell)
Playbook avec les modules Ansible préfaits (apt, service, ...)
Playbook structuré sous forme de rôle Ansible
Prérequis
Une machine Ubuntu locale (control node) sur laquelle vous avez un accès sudo
Une instance EC2 Ubuntu déjà créée sur AWS, démarrée et accessible
Une clé SSH déjà présente sur la machine locale et autorisée à se connecter à l'EC2 (ex : ~/.ssh/ma-cle.pem)
Le Security Group de l'EC2 autorise :
le port 22 (SSH) depuis votre IP
le port 80 (HTTP) depuis n'importe où (0.0.0.0/0), pour vérifier Nginx à la fin
L'adresse IP publique (ou le DNS public) de l'EC2
💡 Vérifiez avant de commencer que la connexion SSH manuelle fonctionne :

ssh -i ~/.ssh/ma-cle.pem ubuntu@<IP_PUBLIQUE_EC2>
Partie 0 — Installation d'Ansible en local
Sur votre machine Ubuntu :

sudo apt update
sudo apt install -y ansible
ansible --version
Créez un dossier de travail pour le TP :

mkdir ~/tp-ansible && cd ~/tp-ansible
Partie 1 — Configuration de l'inventaire
Créez le fichier inventory.ini :

[web]
ec2-nginx ansible_host=<IP_PUBLIQUE_EC2> ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/ma-cle.pem
Remplacez <IP_PUBLIQUE_EC2> par l'IP publique (ou le DNS public) de votre instance, et adaptez le chemin de la clé si besoin.

Testez la connectivité Ansible :

ansible web -i inventory.ini -m ping
Résultat attendu :

ec2-nginx | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
Si vous avez une erreur de type "Host key checking", vous pouvez soit accepter la clé manuellement (ssh une première fois), soit désactiver la vérification en créant un fichier ansible.cfg :

[defaults]
host_key_checking = False
inventory = inventory.ini
Partie 2 — Playbook simple SANS module préfait
Ici, on installe Nginx uniquement en exécutant des commandes shell brutes, sans utiliser les modules dédiés d'Ansible (apt, service, nginx...). On se limite volontairement aux modules génériques command/shell.

Créez 01-playbook-brut.yml :

---
- name: Installer nginx via des commandes shell (sans module préfait)
  hosts: web
  become: yes

  tasks:
    - name: Mettre à jour le cache apt
      command: apt-get update

    - name: Installer nginx
      command: apt-get install -y nginx

    - name: Démarrer et activer le service nginx
      command: systemctl enable --now nginx
Exécution :

ansible-playbook -i inventory.ini 01-playbook-brut.yml
⚠️ Limite pédagogique : ce playbook n'est pas idempotent. Si vous le relancez, apt-get install réessaiera à chaque fois de tout réinstaller, et Ansible affichera systématiquement changed même si rien n'a réellement changé. C'est exactement le problème que les modules préfaits résolvent.

Partie 3 — Playbook avec les modules Ansible préfaits
On refait la même chose, mais avec les modules officiels : apt et service. Ces modules sont idempotents : ils ne font une action que si c'est nécessaire.

Créez 02-playbook-modules.yml :

---
- name: Installer nginx via les modules Ansible
  hosts: web
  become: yes

  tasks:
    - name: Mettre à jour le cache apt
      apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Installer nginx
      apt:
        name: nginx
        state: present

    - name: Démarrer et activer le service nginx
      service:
        name: nginx
        state: started
        enabled: yes
Exécution :

ansible-playbook -i inventory.ini 02-playbook-modules.yml
Relancez la commande une seconde fois : vous devriez constater que tout passe en ok et non plus en changed, preuve de l'idempotence.

Auteur Flo Tch - tp-ansible-nginx-ec2 / DevOps
