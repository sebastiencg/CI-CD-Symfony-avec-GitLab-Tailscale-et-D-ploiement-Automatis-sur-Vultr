# CI-CD-Symfony-avec-GitLab-Tailscale-et-D-ploiement-Automatis-sur-Vultr

Mise en place d’un pipeline CI/CD complet pour un projet Symfony avec GitLab, Tailscale et déploiement automatisé sur un serveur Vultr

Dans ce projet, j’ai mis en place une chaîne CI/CD complète pour mon site portfolio Symfony. L’objectif était d’automatiser entièrement le processus de test, de validation, puis de déploiement sur un serveur de production isolé, tout en assurant un haut niveau de sécurité réseau.

Ce travail a couvert plusieurs aspects :

intégration continue (CI) avec exécution des tests unitaires

déploiement continu (CD) automatisé via SSH

sécurisation de l’infrastructure avec Tailscale

gestion fine des ACL pour isoler un serveur public (Vultr)

et résolution de plusieurs contraintes liées à Symfony, GitLab Runner et aux dépendances PHP.

Voici le déroulé complet de ce que j’ai mis en place.

1. Mise en place de l’intégration continue (CI)
1.1. Image CI personnalisée

J’ai commencé par configurer GitLab CI avec une image Docker basée sur PHP 8.3.
J'y ai ajouté les extensions requises par Symfony et Doctrine (intl, pdo_sqlite…).

1.2. Installation des dépendances & gestion des environnements

J’ai configuré l’étape de test pour :

installer Composer

installer les dépendances du projet

utiliser une base SQLite en mémoire pour les tests

générer le schéma Doctrine pour l’environnement test

et lancer PHPUnit

Cette étape garantit que chaque commit est testé automatiquement avant d’être accepté.

1.3. Correction des tests existants

Les tests fournis par défaut par Symfony n’étaient plus valides, notamment à cause de changements dans les routes (préfixes /admin/post) et des règles de sécurité (redirection vers /login).
J’ai donc adapté les URLs utilisées dans les tests pour les aligner sur l’application actuelle.

Résultat : les tests unitaires passent désormais systématiquement lors de chaque pipeline.

2. Mise en place de Tailscale pour sécuriser la chaîne CI/CD
2.1. Objectif sécurité

Mon GitLab est auto-hébergé et accessible via un tunnel Cloudflare.
En revanche, mon serveur de production Vultr est exposé publiquement et n’a pas accès au réseau interne.

Je voulais :

que GitLab Runner puisse déployer vers Vultr,

sans ouvrir de ports publics,

et en isolant Vultr des autres machines du réseau VPN.

2.2. Mise en réseau Zero-Trust avec Tailscale

J’ai intégré tous les serveurs dans un réseau privé VPN WireGuard grâce à Tailscale.

Trois machines sont concernées :

Machine	Adresse Tailscale	Rôle
serveur-principale	100.106.x.x	GitLab auto-hébergé
PC (développement)	100.87.x.x	Machine de travail
Vultr	100.85.x.x	Serveur de production
2.3. Mise en place des ACL Tailscale

J’ai configuré une règle Zero-Trust stricte :

seule GitLab peut communiquer avec le serveur Vultr

Vultr ne peut communiquer avec rien d’autre

aucune autre machine du réseau n’a accès à Vultr

tout le reste du réseau fonctionne normalement

Exemple d’ACL simplifiée utilisée :

{
  "tagOwners": {
    "tag:vultr": ["autogroup:admin"]
  },

  "grants": [
    {
      "src": ["100.106.224.109"], 
      "dst": ["tag:vultr"],
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


Résultat :
➡️ Vultr est totalement isolé, accessible uniquement par GitLab pour le déploiement.
➡️ Sécurité réseau maximale sans ouvrir le moindre port au public.

3. Mise en place du déploiement continu (CD)
3.1. Méthode retenue

J’ai choisi un déploiement en SSH via GitLab Runner :

GitLab exécute les tests

Si tout est OK, GitLab ouvre une session SSH vers Vultr

Vultr récupère la dernière version du code depuis GitLab

Symfony reconstruit le cache et met à jour le schéma Doctrine

Droits corrigés sur var/ pour permettre l’écriture par PHP-FPM

3.2. Déploiement sécurisé avec clés SSH

J’ai créé une clé privée que GitLab utilise pour se connecter à Vultr.
La clé publique a été installée dans ~/.ssh/authorized_keys du serveur Vultr.

Là encore : aucune ouverture de port supplémentaire, tout passe via Tailscale.

3.3. Pipeline GitLab complet

Le pipeline final comporte ces étapes :

test (CI)

deploy_production (CD)

Et utilise cette structure :

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

3.4. Gestion du cache Symfony en production

Lors des premiers déploiements, Symfony affichait :

Unable to write in the cache directory (...)


Ce problème venait du fait que le cache était créé par root pendant le déploiement.

Solution :
➡️ Ajout automatique d’un chown -R www-data:www-data var/ dans le script.

4. Résultat final

Grâce à toute cette chaîne CI/CD :

✔️ Chaque commit est testé automatiquement

Tests unitaires exécutés via SQLite en environnement isolé.

✔️ Déploiement automatique en production dès que la branche main est mise à jour

GitLab se charge du git fetch, de l’installation, du cache, et de la migration.

✔️ Vultr est protégé dans un environnement Zero-Trust

Aucun port ouvert.
Accessible uniquement via Tailscale par mon GitLab.

✔️ Gestion propre des dépendances, cache et droit fichiers Symfony

Le site reste stable et opérationnel à chaque déploiement.

✔️ Plus aucune intervention manuelle sur le serveur

Le déploiement est fiable, reproductible et sécurisé.

Conclusion

Ce projet m’a permis de mettre en place une stack CI/CD professionnelle complète autour de Symfony :

orchestration GitLab CI/CD

Test Unitaires automatisés

Zero-Trust Networking avec Tailscale

ACL avancées pour isoler les serveurs sensibles

Déploiement sécurisé en SSH sans aucune exposition réseau

Pipeline propre, robuste et maintenable

Résolution des contraintes Symfony (cache, migrations, droits, extensions PHP)

C’est désormais une infrastructure solide, sécurisée, automatisée et scalable, que je peux réutiliser sur d’autres projets (Symfony, Node, ou autres).
