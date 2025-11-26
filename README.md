
# CI/CD complet pour un projet Symfony avec GitLab, Tailscale et Vultr

![status](https://img.shields.io/badge/status-production-green)
![symfony](https://img.shields.io/badge/Symfony-7.x-black)
![gitlab](https://img.shields.io/badge/GitLab-CI%2FCD-orange)
![tailscale](https://img.shields.io/badge/Tailscale-Zero--Trust-blue)
![php](https://img.shields.io/badge/PHP-8.3-777BB4)
![license](https://img.shields.io/badge/license-MIT-lightgrey)

Ce dépôt documente la mise en place d’un pipeline CI/CD complet et sécurisé pour un projet Symfony,
avec tests unitaires, déploiement automatisé, et infrastructure Zero-Trust grâce à Tailscale.

---

## Table des matières

1. [Architecture générale](#architecture-générale)
2. [Intégration Continue (CI)](#1-intégration-continue-ci)  
   1. [Image CI personnalisée](#11-image-ci-personnalisée)  
   2. [Installation des dépendances & environnement test](#12-installation-des-dépendances--gestion-de-lenvironnement-test)  
   3. [Correction des tests Symfony](#13-correction-des-tests-existants)  
3. [Sécurisation réseau avec Tailscale](#2-sécurisation-via-tailscale)  
   1. [Objectifs](#21-objectif)  
   2. [Mise en réseau Zero-Trust](#22-mise-en-réseau-wireguard-zero-trust)  
   3. [ACL Tailscale](#23-acl-tailscale)  
4. [Déploiement Continu (CD)](#3-déploiement-continu-cd)  
   1. [Méthode de déploiement](#31-méthode-de-déploiement)  
   2. [Connexion SSH via Tailscale](#32-connexion-ssh-sécurisée)  
   3. [Pipeline GitLab complet](#33-exemple-de-job-gitlab)  
   4. [Gestion du cache Symfony](#34-gestion-du-cache-symfony)  
5. [Résultats](#4-résultats)  
6. [Conclusion](#conclusion)

---

## Architecture générale

Schéma simplifié de l'infrastructure CI/CD :

```text
             ┌──────────────────────────────┐
             │      Machine locale (DEV)    │
             │    100.87.x.x (Tailscale)    │
             └───────────────▲──────────────┘
                             │
                             │
                  Tailscale Mesh VPN
                             │
                             ▼
    ┌────────────────────────────────────────────┐
    │         Serveur GitLab Auto-Hébergé        │
    │       100.106.x.x (Tailscale)              │
    │  - GitLab CI/CD                            │
    │  - GitLab Runner                           │
    └───────────────▲────────────────────────────┘
                    │  SSH (via Tailscale)
                    │  Zero-Trust Rule
                    ▼
    ┌────────────────────────────────────────────┐
    │         Serveur Vultr (PRODUCTION)         │
    │          100.85.x.x (Tailscale)            │
    │  - Symfony en prod                         │
    │  - Aucun port public ouvert                │
    │  - Accès uniquement depuis GitLab          │
    └────────────────────────────────────────────┘
````

---

## 1. Intégration Continue (CI)

### 1.1. Image CI personnalisée

Une image Docker custom basée sur **PHP 8.3** est utilisée avec :

* `intl`
* `pdo_sqlite`
* `zip`
* autres extensions nécessaires à Symfony

### 1.2. Installation des dépendances & gestion de l'environnement test

La CI :

* installe Composer
* installe les dépendances
* crée une base SQLite en mémoire
* génère le schéma Doctrine
* exécute PHPUnit

Cela garantit des tests reproductibles à chaque commit.

### 1.3. Correction des tests existants

Les tests de base Symfony échouaient (routes modifiées, redirections `/login`).
Les URLs ont été mises à jour pour refléter l’état actuel de l’application.

---

## 2. Sécurisation via Tailscale

### 2.1. Objectif

* GitLab doit déployer vers Vultr
* Aucun port ne doit être ouvert
* Vultr doit être totalement isolé
* Sécurité "Zero-Trust"

### 2.2. Mise en réseau WireGuard Zero-Trust

| Machine            | Adresse Tailscale | Rôle          |
| ------------------ | ----------------- | ------------- |
| serveur-principale | 100.106.x.x       | GitLab        |
| PC                 | 100.87.x.x        | Développement |
| Vultr              | 100.85.x.x        | Production    |

### 2.3. ACL Tailscale

Extrait d’ACL :

```json
{
  "tagOwners": {
    "tag:tag": ["autogroup:admin"]
  },
  "grants": [
    {
      "src": ["100.106.xx.xx"],
      "dst": ["tag:tag"],
      "ip": ["*"]
    }
  ],
  "ssh": [
    {
      "action": "check",
      "src": ["autogroup:member"],
      "dst": ["autogroup:self"],
      "users": ["autogroup:nonroot", "root"]
    }
  ]
}
```

**Effet :**

* GitLab → Vultr : autorisé
* Toute autre machine → Vultr : interdit
* Vultr → monde extérieur : interdit

---

## 3. Déploiement Continu (CD)

### 3.1. Méthode de déploiement

1. GitLab exécute les tests.
2. Si OK : SSH via Tailscale vers Vultr.
3. Mise à jour du dépôt Git.
4. Installation Composer optimisée.
5. Mise à jour Doctrine.
6. Clear cache Symfony.
7. Correction des permissions.

### 3.2. Connexion SSH sécurisée

* Clé privée : stockée dans GitLab.
* Clé publique : dans `~/.ssh/authorized_keys` sur Vultr.
* Pas d’ouverture de port (SSH Tailscale uniquement).

### 3.3. Exemple de job GitLab

```yaml
deploy_production:
  stage: deploy
  only:
    - main
  script:
    - ssh root@100.85.xx.xx "
        set -e;
        cd /home/portfolio-symfony;

        git fetch gitlab;
        git reset --hard gitlab/main;

        composer install --no-dev --optimize-autoloader;

        php bin/console doctrine:schema:update --force --env=prod;
        php bin/console cache:clear --env=prod;

        chown -R www-data:www-data var;
      "
```

### 3.4. Gestion du cache Symfony

Erreur rencontrée :

```text
Unable to write in the cache directory (...)
```

Solution :

```bash
chown -R www-data:www-data var/
```

---

## 4. Résultats

* Tests unitaires automatiques
* Déploiement instantané sur `main`
* Vultr totalement isolé via Zero-Trust
* Aucune exposition réseau
* Gestion propre des dépendances et du cache
* Déploiement 100% automatisé, sans intervention humaine

---

## Conclusion

Cette stack CI/CD fournit :

* une infrastructure sécurisée et automatisée
* un déploiement reproductible, rapide et fiable
* un réseau Zero-Trust
* une intégration propre avec Symfony et GitLab

Cette architecture est réutilisable pour tout projet Symfony, Node.js, ou autres.
