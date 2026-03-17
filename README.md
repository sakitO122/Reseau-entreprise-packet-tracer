# Simulation Réseau d'Entreprise — Cisco Packet Tracer

> Architecture réseau d'entreprise multi-sites complète : OSPFv2, HSRP, EtherChannel, VLANs, sécurité L2/L3, VPN GRE/IPSec

---

## Présentation

Ce projet simule un réseau d'entreprise professionnel sous Cisco Packet Tracer.
Il couvre l'ensemble des technologies utilisées en production : routage dynamique, haute disponibilité,
segmentation par VLANs, sécurité couche 2 et 3, et interconnexion de sites distants via VPN.

---

## Architecture

```
                        Internet (ISP - 2911)
                       /                    \
              203.0.113.1                 200.0.0.2
              R0-HQ (2911)  ←GRE/IPSec→  R-Branch (2911)
              /         \                      |
        10.0.0.1      10.0.1.1            172.16.0.1
            |               |            SW-Branch (2960)
       SW-Core-A ══Po1══ SW-Core-B         PC-B1 · PC-B2
      (3650-24PS)  LACP  (3650-24PS)
      HSRP Active       HSRP Standby
           |                  |
     ┌─────┴────┐       ┌─────┴────┐
   SW-A1      SW-A2   SW-A3      SW-A4
  VLAN 10   VLAN 20  VLAN 30   VLAN 40
  Direction   Dev      RH      Serveurs
```

---

## Plan d'adressage IP

### VLANs HQ

| VLAN | Nom | Réseau | Gateway HSRP | DHCP Range |
|------|-----|--------|--------------|------------|
| 10 | Direction | 10.0.10.0/24 | 10.0.10.1 | .100 → .150 |
| 20 | Développement | 10.0.20.0/24 | 10.0.20.1 | .100 → .200 |
| 30 | RH | 10.0.30.0/24 | 10.0.30.1 | .100 → .130 |
| 40 | Serveurs | 10.0.40.0/24 | 10.0.40.1 | Statique |
| 99 | Management | 10.0.99.0/24 | 10.0.99.1 | .10 → .20 |

### Liens inter-équipements

| Lien | Réseau |
|------|--------|
| R0-HQ ↔ SW-Core-A | 10.0.0.0/30 |
| R0-HQ ↔ SW-Core-B | 10.0.1.0/30 |
| Tunnel GRE HQ ↔ Branch | 192.168.100.0/30 |
| Branch LAN | 172.16.0.0/24 |
| WAN HQ | 203.0.113.0/24 |
| WAN Branch | 200.0.0.0/24 |

### Serveurs (statiques)

| Serveur | IP | Rôle |
|---------|-----|------|
| Serveur-1 | 10.0.40.10 | HTTP / Web interne |
| Serveur-2 | 10.0.40.12 | DHCP + DNS |

---

## Équipements Packet Tracer

| Équipement | Modèle | Rôle |
|---|---|---|
| R0-HQ | Cisco 2911 | Routeur central OSPF + NAT + VPN |
| R-Branch | Cisco 2911 | Routeur site distant + VPN |
| ISP | Cisco 2911 | Simulation lien Internet WAN |
| SW-Core-A | Cisco 3650-24PS | Core L3 — HSRP Active |
| SW-Core-B | Cisco 3650-24PS | Core L3 — HSRP Standby |
| SW-A1 | Cisco 2960-24TT | Access VLAN 10 Direction |
| SW-A2 | Cisco 2960-24TT | Access VLAN 20 Dev |
| SW-A3 | Cisco 2960-24TT | Access VLAN 30 RH |
| SW-A4 | Cisco 2960-24TT | Access VLAN 40 Serveurs |
| SW-MGMT | Cisco 2960-24TT | Access VLAN 99 Management |
| SW-Branch | Cisco 2960-24TT | Switch site distant |
| Serveur-1 | Generic Server | HTTP interne |
| Serveur-2 | Generic Server | DHCP + DNS |

---

## Technologies implémentées

### Routage dynamique — OSPFv2
- Area 0 unique, router-id explicite sur chaque équipement
- Load balancing dual path (SW-Core-A + SW-Core-B simultanément)
- Propagation route par défaut : `default-information originate`
- `passive-interface` sur tous les SVIs hôtes

### Haute disponibilité
- **HSRP** Active/Standby — failover < 4 secondes (timers 1s/3s)
- **EtherChannel LACP** Po1 — agrégation 2x GigE = 2 Gbps entre Core switches
- Preempt activé sur SW-Core-A (priorité 110 vs 100)

### Sécurité couche 2
- **Port Security** sticky — max 2 MACs, violation restrict
- **DHCP Snooping** — filtre serveurs DHCP non autorisés
- **DAI** (Dynamic ARP Inspection) — anti ARP spoofing / MITM
- **STP PortFast + BPDUGuard** sur tous les ports Access

### Sécurité couche 3 — ACL inter-VLAN

| VLAN | Accès autorisé | Accès bloqué |
|------|---------------|--------------|
| Direction (10) | Serveurs · RH · Internet | Dev · Management |
| Dev (20) | Serveurs · Internet | Direction · RH · Management |
| RH (30) | Serveurs · DNS · Internet | Direction · Dev · Management |
| Serveurs (40) | Répond uniquement | N'initie aucune connexion |
| Management (99) | Accès total (admins) | Inaccessible depuis autres VLANs |

### VPN site-à-site
- Tunnel **GRE** point-à-point HQ ↔ Branch
- Chiffrement **IPSec AES-256 / SHA / IKEv1 / DH Group 2**
- Clé pré-partagée : `SecretVPN`
- Routage statique dans le tunnel

### Services réseau
- **DHCP** centralisé sur Serveur-2 avec `ip helper-address` par VLAN
- **DNS** interne - résolution `entreprise.local`
- **NAT overload (PAT)** pour accès Internet depuis tous les VLANs
- **SSH** restreint au VLAN 99 via ACL-SSH-MGMT

---

## Politique de sécurité inter-VLAN

```
ACL-DIR-POLICY  (VLAN 10) : permit Serveurs · permit RH · deny Dev · deny Mgmt · permit any
ACL-DEV-POLICY  (VLAN 20) : permit Serveurs · deny Direction · deny RH · deny Mgmt · permit any
ACL-RH-POLICY   (VLAN 30) : permit DNS/HTTP Serveurs · deny Dev · deny Direction · deny Mgmt · permit any
ACL-SRV-POLICY  (VLAN 40) : permit established · permit HTTP/HTTPS/DNS entrant · deny vers VLANs users
ACL-MGMT-POLICY (VLAN 99) : permit tout depuis 10.0.99.0/24 · deny tout vers 10.0.99.0/24
ACL-SSH-MGMT    (VTY)     : permit 10.0.99.0/24 · deny any
```

---

## Structure du dépôt

```
reseau-entreprise-packet-tracer/
├── README.md                          # Ce fichier
├── topology/
│   └── reseau_entreprise.pkt          # Fichier Packet Tracer
├── configs/
│   ├── R0-HQ.txt                      # Config complète R0
│   ├── R-Branch.txt                   # Config complète R-Branch
│   ├── ISP.txt                        # Config ISP
│   ├── SW-Core-A.txt                  # Config SW-Core-A
│   ├── SW-Core-B.txt                  # Config SW-Core-B
│   ├── SW-A1.txt                      # Config SW-A1
│   ├── SW-A2.txt                      # Config SW-A2
│   ├── SW-A3.txt                      # Config SW-A3
│   ├── SW-A4.txt                      # Config SW-A4
│   ├── SW-MGMT.txt                    # Config SW-MGMT
│   └── SW-Branch.txt                  # Config SW-Branch
├── rapport/
│   └── rapport_projet.docx            # Rapport détaillé du projet
└── screenshots/
    ├── topologie_generale.png
    ├── ospf_neighbors.png
    ├── hsrp_active.png
    ├── etherchannel_summary.png
    ├── acl_tests.png
    ├── vpn_tunnel.png
    └── ssh_management.png
```

---

## Résultats des tests

| Test | Commande | Résultat |
|------|---------|---------|
| Connectivité R0 ↔ Core-A | `ping 10.0.0.2` |  100% |
| Connectivité R0 ↔ Core-B | `ping 10.0.1.2` |  100% |
| Connectivité R0 ↔ Branch | `ping 172.16.0.1` |  100% |
| Tunnel GRE | `ping 192.168.100.2` |  100% |
| OSPF convergence | `show ip ospf neighbor` |  FULL/DR |
| OSPF load balancing | `show ip route ospf` |  dual path |
| HSRP Core-A Active | `show standby brief` |  Active prio 110 |
| HSRP failover | Extinction Core-A | < 4 secondes |
| EtherChannel Po1 | `show etherchannel summary` |  SU — 2 ports P |
| DHCP VLAN 10 | PC Direction → DHCP |  10.0.10.10x |
| DHCP VLAN 20 | PC Dev → DHCP |  10.0.20.10x |
| DHCP VLAN 30 | PC RH → DHCP |  10.0.30.10x |
| ACL RH bloque Dev | ping 10.0.20.1 depuis RH |  Bloqué |
| ACL Dev bloque Direction | ping 10.0.10.1 depuis Dev |  Bloqué |
| SSH depuis VLAN 99 | ssh admin@10.0.99.2 | Autorisé |
| SSH depuis VLAN 10 | ssh admin@10.0.99.2 |  Refusé |
| Port Security | show port-security | Actif sticky |

---

## Prérequis

- Cisco Packet Tracer **8.2 minimum**
- Package **securityk9** activé sur les 2911 (R0-HQ et R-Branch)
- Package **datak9** activé pour OSPF avancé

---

## Ordre de déploiement recommandé

1. ISP -interfaces + routes statiques
2. R0-HQ - interfaces, OSPF, NAT, GRE, IPSec
3. SW-Core-A - VLANs, SVIs, HSRP Active, EtherChannel, OSPF, ACL
4. SW-Core-B - VLANs, SVIs, HSRP Standby, EtherChannel, OSPF
5. SW-A1 à SW-A4 - VLANs, Port Security, DHCP Snooping, DAI
6. SW-MGMT - VLAN 99, trunk vers Core-B
7. Serveur-2 - pools DHCP (VLAN 10/20/30/99), DNS, HTTP
8. R-Branch - interfaces, OSPF, GRE, IPSec
9. SW-Branch - configuration basique
10. PCs - DHCP automatique / statique pour serveurs

---

## Auteur

Projet réalisé par SAIH PIELY URIEL LOIC
Simulation Cisco Packet Tracer - Architecture entreprise complète