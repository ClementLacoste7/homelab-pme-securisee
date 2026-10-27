# Conception d'une infrastructure PME virtualisée et sécurisée

Projet personnel de homelab reproduisant l'infrastructure d'une petite entreprise, avec segmentation réseau, annuaire Active Directory, durcissement selon les recommandations de l'ANSSI, supervision par SIEM, tests d'intrusion internes et automatisation.

L'objectif est de pratiquer de bout en bout les métiers de l'administration réseau, de l'infrastructure et de la cybersécurité, sur un environnement complet et isolé.

## Technologies

- Virtualisation : Proxmox VE 9
- Pare-feu et routage : pfSense CE
- Annuaire : Windows Server 2025, Active Directory, GPO
- Poste client : Windows 11
- Supervision (SIEM) : Wazuh sur Ubuntu Server 24.04
- Tests d'intrusion : Kali Linux (nmap, netexec, kerbrute)
- Automatisation : Ansible
- Services d'appui : AdGuard Home, Vaultwarden, Homepage, Tailscale

## Architecture réseau

Le réseau est segmenté en VLAN par pfSense, pour isoler les usages.

| VLAN | Usage | Réseau | Machines |
|------|-------|--------|----------|
| 10 | Administration | 10.10.10.0/24 | pfSense LAN |
| 20 | Serveurs | 10.10.20.0/24 | DC01 (10.10.20.10), Wazuh (10.10.20.20) |
| 30 | Employés | 10.10.30.0/24 | PC01, Kali (poste d'attaque) |

Domaine Active Directory : corp.lab.lan (NetBIOS CORP).

Inventaire des machines virtuelles et conteneurs :

| ID | Nom | Rôle |
|----|-----|------|
| 100 | adguard | Filtrage DNS |
| 101 | homepage | Tableau de bord des services |
| 102 | vaultwarden | Gestionnaire de mots de passe |
| 103 | pfsense | Pare-feu et routage inter-VLAN |
| 104 | DC01 | Contrôleur de domaine Active Directory |
| 105 | PC01 | Poste client joint au domaine |
| 106 | wazuh | Serveur SIEM |
| 107 | kali | Poste de test d'intrusion |

## Phases du projet

### Phase 1 : socle et réseau
Installation de Proxmox, mise en place de pfSense, création des VLAN, segmentation du réseau et règles de filtrage inter-VLAN basées sur le principe du moindre privilège.

### Phase 2 : Active Directory
Promotion du contrôleur de domaine, création de la structure des unités d'organisation (OU), des utilisateurs et des groupes par service. Mise en place de GPO (blocage du Panneau de configuration, verrouillage de session, lecteurs réseau mappés par groupe via ciblage au niveau de l'élément). Jonction du poste client au domaine.

### Phase 3 : durcissement ANSSI
- Politique de mot de passe renforcée : 12 caractères minimum, historique de 24, verrouillage après 5 échecs.
- Désactivation du compte Administrateur intégré et création d'un compte d'administration dédié.
- Restriction du pare-feu : les postes employés n'atteignent le contrôleur de domaine que sur les ports strictement nécessaires à l'Active Directory.
- Activation de l'audit des connexions et de la gestion des comptes.

### Phase 4 : supervision et détection
Déploiement du SIEM Wazuh et installation des agents sur le contrôleur de domaine et le poste client. Génération d'attaques depuis Kali et observation de la détection.

Deux enseignements clés :
- Un scan de ports furtif (nmap en mode SYN) n'est pas détecté par un SIEM basé uniquement sur les journaux Windows. C'est un angle mort à combler par une sonde réseau (IDS) ou les journaux du pare-feu.
- Une attaque par pulvérisation de mots de passe sur Kerberos (password spraying, technique MITRE ATT&CK T1110.003) génère des événements 4771 sur le contrôleur de domaine, détectés et corrélés par Wazuh.

### Phase 5 : automatisation
Mise en place d'Ansible comme outil d'automatisation de la configuration. Rédaction d'un playbook de durcissement idempotent (installation et activation de fail2ban). Documentation et publication du projet.

## Automatisation Ansible

Exemple de playbook de durcissement appliqué au serveur de supervision :

```yaml
---
- hosts: localhost
  connection: local
  become: true
  tasks:
    - name: Mettre a jour le cache apt
      apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Installer fail2ban
      apt:
        name: fail2ban
        state: present

    - name: Activer et demarrer fail2ban
      service:
        name: fail2ban
        state: started
        enabled: true
```

L'option cache_valid_time rend la mise à jour du cache idempotente : elle ne se relance que si le cache date de plus d'une heure.

## Problèmes rencontrés et solutions

Cette section documente les difficultés réelles rencontrées pendant le projet et la manière dont elles ont été résolues. C'est le coeur de l'apprentissage.

**Mauvaise image d'installation de Windows Server.** La VM ne démarrait pas. L'ISO téléchargée était une version de composants (LOF Packages) et non une version installable. Solution : télécharger l'ISO d'évaluation (Eval).

**Dépôts Proxmox en erreur 401.** Les mises à jour échouaient. Les dépôts Enterprise exigent un abonnement. Solution : désactiver les dépôts Enterprise et activer le dépôt no-subscription.

**Accès à pfSense impossible depuis le WAN.** Le pare-feu bloque par défaut l'administration côté WAN. Solution : administrer pfSense depuis l'interface LAN.

**Règles de pare-feu sans effet.** Deux causes distinctes : un mauvais ordre des règles (elles sont évaluées de haut en bas, première correspondance) et des modifications non appliquées (bouton Apply Changes oublié). Les compteurs de correspondance des règles ont permis de diagnostiquer.

**Poste client incapable de joindre le domaine.** Le DNS du poste ne pointait pas vers le contrôleur de domaine, donc les enregistrements SRV du domaine étaient introuvables. Solution : distribuer l'adresse du contrôleur comme serveur DNS via le DHCP du VLAN.

**Erreur Accès refusé à la jonction au domaine.** Le terminal n'était pas élevé. Le contrôle de compte (UAC) limite les droits même pour un administrateur. Solution : lancer le terminal en tant qu'administrateur.

**Alertes absentes du tableau de bord Wazuh.** Un filtre d'agent épinglé masquait les résultats. Le diagnostic de bout en bout (source, transport, affichage) a permis de confirmer que les événements arrivaient bien, et de corriger l'affichage.

**Scan nmap signalant l'hôte comme éteint.** Le pare-feu bloque l'ICMP, donc nmap croyait la cible hors ligne. Solution : utiliser l'option -Pn pour ignorer la découverte par ping.

**Timeouts lors du test SMB.** Les tentatives d'authentification SMB expiraient. Solution : basculer sur un test Kerberos (kerbrute, port 88), fiable et générant des événements d'audit exploitables.

**Ansible incapable d'atteindre une cible sur un autre réseau.** Le conteneur cible utilisait la box comme passerelle, provoquant un routage asymétrique. Solution : retenir une cible joignable et cohérente avec la topologie.

**Saturation du stockage Proxmox.** La partition des fichiers (local) se remplissait à cause de sauvegardes trop lourdes. Solution : exclure le poste client des sauvegardes, limiter la rétention à deux versions, et faire le ménage des images et sauvegardes obsolètes.

## Compétences mises en oeuvre

- Virtualisation et administration d'un hyperviseur
- Segmentation réseau et filtrage par VLAN
- Administration d'un annuaire Active Directory et stratégies de groupe
- Durcissement d'un système d'information selon l'ANSSI
- Mise en place et exploitation d'un SIEM
- Tests d'intrusion internes et lecture des traces côté défense
- Automatisation de la configuration par Ansible
