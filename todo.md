# TODO — Architecture Web distribuée avec HAProxy, Apache, PHP/CI4 et PostgreSQL

## 1. Présentation du projet

### Objectif général

Mettre en place une architecture Web distribuée permettant de :

- Distribuer les requêtes HTTP entre plusieurs serveurs Apache.
- Assurer une meilleure disponibilité de l'application.
- Utiliser PostgreSQL comme système de gestion de données.
- Répliquer automatiquement les données du serveur PostgreSQL principal vers un serveur secondaire.
- Séparer les opérations de lecture et d'écriture.
- Tester le comportement de l'architecture en cas de panne.

### Technologies utilisées (versions et choix réels du projet)

| Technologie | Rôle | État actuel |
|---|---|---|
| Ubuntu | Système d'exploitation | À vérifier |
| HAProxy | Reverse proxy et load balancing | À installer |
| Apache2 | Serveur Web (2 instances, 8081 et 8082) | À installer |
| PHP 8.3 (CLI) | Langage backend | Present (`composer.json` exige `^8.2`) |
| CodeIgniter 4 | Framework MVC | Déjà installé à la racine du dépôt |
| Composer | Gestion des dépendances PHP | À vérifier (`composer --version`) |
| PostgreSQL | SGBD, Primary (5432) et Standby (5433) | À installer/configurer |
| Extension PHP `pgsql` | Pilote `Postgre` de CI4 (`pg_connect`) | À vérifier (`php -m`) |
| Spark (CLI CI4) | Génération, migrations, seeders, cache | Disponible (`./spark`) |
| PHPUnit | Tests unitaires CI4 | Disponible (`composer test`) |
| cURL | Tests HTTP | À installer |

> **Correction importante :** le projet n'utilise pas PDO. Le pilote `CodeIgniter\Database\Postgre\Connection` utilise les fonctions `pg_*` de l'extension `pgsql`. Le bon paquet à installer est donc `php-pgsql` (et non `php-pdo-pgsql` comme unique dépendance). Les dépendances Composer déjà requises sont `ext-intl`, `ext-mbstring`, `laminas/laminas-escaper` et `psr/log`.

---

# 2. État actuel du dépôt (point de départ)

Le dossier `CI4/` **est déjà un projet CodeIgniter 4 complet** : il ne faut pas créer de sous-dossier `application/` ni réinstaller le framework.

## 2.1 Ce qui existe déjà

| Élément | Emplacement | Remarque |
|---|---|---|
| Framework CI4 + `system/` | `./system/`, `./spark`, `./preload.php` | Fourni par la distribution CI4 |
| Point d'entrée Web | `public/index.php`, `public/.htaccess` | Le DocumentRoot Apache doit pointer sur `public/` |
| Contrôleur d'accueil | `app/Controllers/Home.php` | Page CI4 par défaut |
| Vue d'accueil | `app/Views/welcome_message.php` | À remplacer par l'accueil de l'application |
| Routes | `app/Config/Routes.php` | Une seule route : `$routes->get('/', 'Home::index');` |
| Configuration app | `app/Config/App.php` | `baseURL = http://localhost:8080/`, `appTimezone = UTC` |
| Configuration DB | `app/Config/Database.php` | Groupe `default` : `Postgre`, `localhost:5432`, base `clust_tp1` ; identifiants **codés en dur** à déplacer dans `.env` |
| Groupe de test | `app/Config/Database.php` (`$tests`) | SQLite3 en mémoire, utilisé automatiquement si `ENVIRONMENT === 'testing'` |
| Migrations / Seeders | `app/Database/Migrations/`, `app/Database/Seeds/` | Vides (`.gitkeep`) |
| Modèles / Filtres / Librairies | `app/Models/`, `app/Filters/`, `app/Libraries/` | Vides (`.gitkeep`) |
| Tests CI4 | `tests/unit/HealthTest.php`, `tests/database/`, `tests/session/` | Tests fournis par CI4 (`CIUnitTestCase`) |
| Environment | `env` (modèle) | Aucun `.env` présent : à créer, **jamais versionné** |
| Composer | `composer.json` | Script `test` = `phpunit`, `phpunit.xml.dist` présent |

## 2.2 Ce qu'il faut corriger en priorité

1. **Déplacer les identifiants PostgreSQL** de `app/Config/Database.php` vers `.env` (le mot de passe est actuellement en clair dans le code).
2. **Définir deux groupes de connexion** `primary` (5432) et `standby` (5433) dans `app/Config/Database.php`, avec `defaultGroup = 'primary'`.
3. **Aligner `app.baseURL`** sur l'URL publique réelle (`http://<IP_HAPROXY>:8080/`) dans `.env` des deux backends ; le `localhost:8080` actuel entrerait en conflit avec le port HAProxy.
4. **Créer le CRUD produits** : modèle, contrôleur, vues, migration, seeder et routes supplémentaires.
5. **Ajouter un endpoint de santé** `/health` utilisé par HAProxy pour tester les backends.
6. **Déployer la même version** du projet sur Apache 1 et Apache 2.

---

# 3. Architecture cible

```text
                        CLIENT
                          |
                          |
                     HAProxy
                     Port 8080
                     /health (check)
                          |
               ---------------------
               |                   |
           Apache 1             Apache 2
           Port 8081            Port 8082
               |                   |
        DocumentRoot           DocumentRoot
          CI4/public/            CI4/public/
               |                   |
               -----------+--------
                          |
                   CI4 : primary / standby
                          |
                   PostgreSQL
                      PRIMARY
                    Port 5432
                          |
                  Streaming Replication
                          |
                   PostgreSQL
                      STANDBY
                    Port 5433
```

### Règles de fonctionnement

- HAProxy reçoit les requêtes HTTP sur le port 8080 et vérifie les backends via `GET /health`.
- HAProxy distribue les requêtes entre Apache 1 (8081) et Apache 2 (8082).
- Les deux Apache hébergent **la même copie** de l'application CI4 (DocumentRoot = `public/`).
- Les opérations INSERT, UPDATE et DELETE sont envoyées au groupe CI4 `primary`.
- Les opérations SELECT qui tolèrent un léger retard sont envoyées au groupe CI4 `standby`.
- Les lectures après écriture et les transactions restent sur `primary`.
- Le Standby reçoit et applique les modifications du Primary, et reste en lecture seule.

---

# 4. Phase 1 — Préparation de l'environnement

## 4.1 Vérification du système

- [ ] Vérifier la version d'Ubuntu.
- [ ] Vérifier les ressources disponibles (RAM, CPU, stockage).
- [ ] Vérifier la connectivité réseau et identifier les adresses IP.
- [ ] Vérifier les ports déjà utilisés (8080 est prévu pour HAProxy, 8081/8082 pour Apache).
- [ ] Vérifier les versions de PHP, Composer et PostgreSQL.
- [ ] Vérifier que l'extension `pgsql` est active : `php -m | grep -i pgsql`.
- [ ] Vérifier que le projet CI4 démarre : `php spark --version`, `composer test`.

Commandes utiles :

```bash
lsb_release -a
free -h
df -h
ip -4 addr
ip route
ss -tulpn
php -v
php -m | grep -Ei 'pgsql|intl|mbstring'
composer --version
php spark --version
composer test
```

## 4.2 Installation des dépendances

- [ ] Installer Apache2.
- [ ] Installer HAProxy.
- [ ] Installer PHP 8.2+ et les extensions `intl`, `mbstring`, `pgsql`, `curl`, `xml`, `zip`.
- [ ] Installer PostgreSQL.
- [ ] Installer Composer si absent.
- [ ] Installer Git et cURL.

```bash
sudo apt update
sudo apt install -y \
    apache2 \
    haproxy \
    php php-cli php-fpm php-pgsql php-curl php-mbstring \
    php-intl php-xml php-zip \
    postgresql \
    git curl composer
```

- [ ] Vérifier les services systemd et les activer au démarrage.

---

# 5. Phase 2 — Mise en place des serveurs Apache

## 5.1 Configuration du premier Apache (8081)

- [ ] Créer les répertoires de travail du serveur 1.
- [ ] Configurer le port d'écoute 8081 (`ports.conf`).
- [ ] Créer le VirtualHost avec `DocumentRoot /srv/tp1/CI4/public`.
- [ ] Autoriser l'accès au dossier `public/` et interdire l'accès à la racine du projet.
- [ ] Donner la propriété de `writable/` à l'utilisateur Apache (`www-data`).
- [ ] Activer le site et tester l'accès HTTP.

## 5.2 Configuration du deuxième Apache (8082)

- [ ] Créer les répertoires de travail du serveur 2.
- [ ] Configurer le port d'écoute 8082.
- [ ] Créer le VirtualHost avec le même `DocumentRoot /srv/tp1/CI4/public`.
- [ ] Appliquer les mêmes permissions, dont `writable/`.
- [ ] Activer le site et tester l'accès HTTP.

## 5.3 Tests Apache

- [ ] Vérifier que les deux serveurs répondent indépendamment.
- [ ] Vérifier les journaux d'erreurs.
- [ ] Vérifier les ports avec `ss`.
- [ ] Vérifier le comportement après redémarrage.

```bash
curl -i http://127.0.0.1:8081/
curl -i http://127.0.0.1:8082/

sudo apache2ctl configtest
sudo systemctl status apache2
```

**Résultat attendu :** deux backends Web indépendants servant la page d'accueil CI4.

---

# 6. Phase 3 — Configuration de HAProxy

## 6.1 Configuration initiale

- [ ] Comprendre les sections `global`, `defaults`, `frontend`, `backend`.
- [ ] Configurer le frontend HTTP sur le port 8080.
- [ ] Déclarer Apache 1 (`127.0.0.1:8081`) et Apache 2 (`127.0.0.1:8082`) dans le backend.
- [ ] Configurer l'algorithme `roundrobin`.
- [ ] Activer les vérifications de disponibilité avec `option httpchk GET /health`.
- [ ] Ajouter un `http-check expect status 200`.

## 6.2 Tests de répartition

- [ ] Tester l'accès à HAProxy : `curl -i http://127.0.0.1:8080/`.
- [ ] Tester plusieurs requêtes et vérifier l'alternance des backends (en-tête `X-Backend` ou logs).
- [ ] Consulter les logs HAProxy.

## 6.3 Tests de disponibilité

- [ ] Arrêter Apache 1, vérifier que Apache 2 sert le trafic, puis redémarrer.
- [ ] Arrêter Apache 2, vérifier le comportement avec Apache 1 uniquement.
- [ ] Arrêter les deux Apache et observer le comportement de HAProxy.

**Résultat attendu :** HAProxy distribue les requêtes et n'envoie plus de trafic vers un backend déclaré indisponible par `/health`.

---

# 7. Phase 4 — Finalisation de l'application CodeIgniter 4

## 7.1 Vérification de l'installation existante

- [ ] Vérifier `php spark --version` et `composer test`.
- [ ] Lancer le serveur local de dev sur un port libre (8083 pour ne pas entrer en conflit avec HAProxy) :

```bash
php spark serve --port 8083
```

- [ ] Vérifier que `GET /` affiche l'accueil actuel (`Home::index`).
- [ ] Ne pas recréer le projet : le code CI4 est déjà à la racine.

## 7.2 Organisation réelle des fichiers

```text
CI4/                                  <- racine du projet (déjà présente)
├── app/
│   ├── Config/
│   │   ├── App.php                   <- baseURL, fuseau horaire (existant)
│   │   ├── Database.php              <- groupes default/primary/standby/tests (à modifier)
│   │   └── Routes.php                <- routes /, /produits/*, /health (à modifier)
│   ├── Controllers/
│   │   ├── BaseController.php        (existant)
│   │   ├── Home.php                  (existant, à adapter)
│   │   ├── Health.php                (à créer : check primary + standby)
│   │   └── ProduitController.php     (à créer : CRUD)
│   ├── Models/
│   │   └── ProduitModel.php          (à créer : $DBGroup = 'primary')
│   ├── Services/
│   │   └── ConnexionService.php      (à créer : primary() / standby())
│   ├── Views/
│   │   ├── welcome_message.php       (existant)
│   │   └── produits/
│   │       ├── liste.php             (à créer)
│   │       ├── ajout.php             (à créer)
│   │       └── modification.php      (à créer)
│   └── Database/
│       ├── Migrations/
│       │   └── 2026-..._create_produits.php   (à créer)
│       └── Seeds/
│           └── ProduitsSeeder.php    (à créer)
├── public/                           <- DocumentRoot Apache (index.php, .htaccess)
├── writable/                         <- droits www-data (logs, cache, sessions)
├── tests/                            <- tests PHPUnit CI4 uniquement
│   ├── unit/
│   ├── database/
│   └── session/
├── system/                           <- framework CI4 (ne pas modifier)
├── env                               <- modèle de variables d'environnement
├── .env                              <- à créer sur chaque serveur, NON versionné
├── composer.json
├── phpunit.xml.dist
└── spark
```

> Les scripts shell de test d'infrastructure ne vont pas dans `tests/` (réservé à PHPUnit) mais dans `scripts/` à la racine du dépôt.

## 7.3 Configuration `.env` et `Database.php`

- [ ] Copier `env` vers `.env` et renseigner les valeurs propres à chaque machine :

```dotenv
app.baseURL = http://<IP_HAPROXY>:8080/
app.appTimezone = UTC

database.default.hostname = 127.0.0.1
database.default.database = clust_tp1
database.default.username = app_user
database.default.password = <mot_de_passe>
database.default.DBDriver = Postgre
database.default.port = 5432

database.primary.hostname = 127.0.0.1
database.primary.database = clust_tp1
database.primary.username = app_user
database.primary.password = <mot_de_passe>
database.primary.DBDriver = Postgre
database.primary.port = 5432

database.standby.hostname = 127.0.0.1
database.standby.database = clust_tp1
database.standby.username = app_user
database.standby.password = <mot_de_passe>
database.standby.DBDriver = Postgre
database.standby.port = 5433
```

- [ ] Dans `app/Config/Database.php`, déclarer les tableaux `$primary` et `$standby` avec `env('database.primary.*')` / `env('database.standby.*')`, faire pointer `$default` vers la même configuration que `primary` (ou utiliser `defaultGroup = 'primary'`), et **supprimer les identifiants en dur**.
- [ ] Conserver le groupe `$tests` (SQLite3 mémoire) et le passage automatique à `tests` en environnement `testing`.
- [ ] Passer `DBDebug` à `false` en production.
- [ ] Ajouter `.env` au `.gitignore`.

## 7.4 CRUD et schéma

- [ ] Créer la migration `app/Database/Migrations/2026-..._create_produits.php` : table `produits` (id, nom, description, prix, quantite, timestamps).
- [ ] Créer le seeder `app/Database/Seeds/ProduitsSeeder.php`.
- [ ] Créer `app/Models/ProduitModel.php` avec `$table = 'produits'`, `$allowedFields`, `$primaryKey = 'id'`, `$useTimestamps = true` et `$DBGroup = 'primary'`.
- [ ] Créer `app/Controllers/ProduitController.php` : `index`, `new`, `create`, `edit`, `update`, `delete` avec validation CI4 (`$this->validate()`).
- [ ] Créer les vues `app/Views/produits/{liste,ajout,modification}.php` avec un layout commun.
- [ ] Ajouter les routes dans `app/Config/Routes.php` :

```php
$routes->get('/', 'Home::index');
$routes->get('health', 'Health::index');
$routes->get('produits', 'ProduitController::index');
$routes->get('produits/new', 'ProduitController::new');
$routes->post('produits', 'ProduitController::create');
$routes->get('produits/(:num)/edit', 'ProduitController::edit/$1');
$routes->post('produits/(:num)/update', 'ProduitController::update/$1');
$routes->post('produits/(:num)/delete', 'ProduitController::delete/$1');
```

- [ ] Appliquer la migration et vérifier le seeder :

```bash
php spark migrate
php spark db:seed ProduitsSeeder
php spark routes
```

## 7.5 Méthode retenue pour séparer lectures et écritures

Principe : **le contrôleur ne choisit jamais directement une connexion** ; il passe par le modèle ou un service.

1. `app/Services/ConnexionService.php` centralise les accès :
   - `primary()` retourne `Database::connect('primary')` ;
   - `standby()` retourne `Database::connect('standby')` ;
   - `produitsEcriture()` retourne `new ProduitModel($this->primary())` ;
   - `produitsLecture()` retourne `new ProduitModel($this->standby())`.
2. Le constructeur de `CodeIgniter\Model` accepte une connexion explicite (`new ProduitModel(Database::connect('standby'))`), ce qui permet de réutiliser le même modèle sur les deux groupes sans dupliquer le code.
3. Routage des opérations :
   - `create()`, `update()`, `delete()` -> `primary()` ;
   - `index()` (liste) -> `standby()` ;
   - `edit()` et toute lecture qui suit une écriture -> `primary()` (cohérence immédiate) ;
   - transactions -> exclusivement `primary()`.
4. Gestion des pannes :
   - si `standby()` lève une `DatabaseException`, bascule transparente sur `primary()` et log l'événement ;
   - si `primary()` est indisponible, renvoyer une erreur 503 explicite (pas de failover automatique avant promotion manuelle du Standby).
5. Ne jamais envoyer toutes les requêtes SELECT vers le Standby : la règle est **explicite par méthode**, pas globale.

## 7.6 Endpoint de santé pour HAProxy

- [ ] Créer `app/Controllers/Health.php` :
  - exécuter `SELECT 1` sur `primary` et sur `standby` ;
  - répondre en JSON avec l'état des deux groupes ;
  - renvoyer le code HTTP `503` si le Primary est indisponible, `200` sinon (le retard de réplication du Standby n'est pas bloquant).
- [ ] Vérifier : `curl -i http://127.0.0.1:8081/health`.

---

# 8. Phase 5 — Déploiement de l'application sur Apache

## 8.1 Déploiement sur Apache 1

- [ ] Copier le projet CI4 dans le répertoire du serveur 1 (ou `git clone` + `git pull`).
- [ ] Vérifier que le VirtualHost pointe sur `public/` et non sur la racine.
- [ ] Exécuter `composer install --no-dev -o`.
- [ ] Créer le `.env` propre au serveur et vérifier `app.baseURL`.
- [ ] Donner les droits d'écriture sur `writable/` à `www-data`.
- [ ] Vider le cache : `php spark cache:clear`.
- [ ] Tester `GET /`, `GET /produits` et `GET /health`.

## 8.2 Déploiement sur Apache 2

- [ ] Déployer **la même version** du code (même commit Git).
- [ ] Appliquer la même configuration Apache et les mêmes droits sur `writable/`.
- [ ] Créer son `.env` (même `baseURL` public, mêmes paramètres de base).
- [ ] Vérifier les connexions `primary` et `standby`.

## 8.3 Tests d'intégration

- [ ] Accéder à l'application via Apache 1, Apache 2, puis HAProxy.
- [ ] Vérifier le CRUD complet via HAProxy.
- [ ] Vérifier que les écritures passent par le Primary et les listes par le Standby (logs PostgreSQL ou `pg_stat_activity`).
- [ ] Vérifier les logs Apache, PHP et CI4 (`writable/logs/`).

---

# 9. Phase 6 — Configuration du PostgreSQL Primary

## 9.1 Préparation

- [ ] Déterminer la version PostgreSQL et le mode d'hébergement (instances, machines ou conteneurs).
- [ ] Définir les ports (5432 Primary, 5433 Standby).
- [ ] Créer la base `clust_tp1`.
- [ ] Créer l'utilisateur applicatif `app_user` (utilisé par CI4, privilèges minimaux).
- [ ] Créer un utilisateur dédié à la réplication (`repl_user`), distinct de `app_user`.

## 9.2 Configuration de la réplication

- [ ] `wal_level = replica`.
- [ ] `max_wal_senders` et `max_replication_slots` > 0.
- [ ] Règles d'accès dans `pg_hba.conf` pour `repl_user` et pour `app_user` depuis les Apache.
- [ ] Créer un slot de réplication physique si nécessaire.
- [ ] Redémarrer PostgreSQL et vérifier les paramètres.

## 9.3 Création du schéma

Le schéma est versionné dans CI4 via `php spark migrate` (voir 7.4) ; l'équivalent SQL reste disponible dans `database/schema.sql` pour documentation.

- [ ] Table `categories` (prévue pour une évolution du CRUD).
- [ ] Table `produits` (obligatoire, créée par la migration CI4).
- [ ] Table `mouvements_stock` (prévue pour une évolution du CRUD).
- [ ] Clés primaires, étrangères, contraintes et index nécessaires.

---

# 10. Phase 7 — Sauvegarde initiale et PostgreSQL Standby

## 10.1 Sauvegarde initiale

- [ ] Vérifier que le Primary fonctionne.
- [ ] Préparer le répertoire de données du Standby.
- [ ] Effectuer la copie initiale avec `pg_basebackup` (slot + WAL).
- [ ] Vérifier l'intégrité de la copie.
- [ ] Configurer le mode Standby (`standby.signal`, `primary_conninfo` sur le port 5432).
- [ ] Démarrer le Standby sur le port 5433.

## 10.2 Réplication en continu

- [ ] Vérifier la connexion entre les instances.
- [ ] Vérifier la réception et l'application des WAL.
- [ ] Contrôler le retard de réplication.
- [ ] Vérifier les statistiques de réplication.

## 10.3 Tests de réplication

- [ ] Ajouter un produit via l'application (écriture Primary), vérifier sa présence sur le Standby.
- [ ] Modifier un produit, vérifier la modification sur le Standby.
- [ ] Supprimer un produit, vérifier la suppression sur le Standby.
- [ ] Vérifier que la liste CI4 (`standby`) reflète bien les changements après réplication.

## 10.4 Tests du mode lecture seule

- [ ] Effectuer un SELECT sur le Standby (via `Database::connect('standby')`).
- [ ] Tenter un INSERT/UPDATE/DELETE sur le Standby.
- [ ] Vérifier le refus des écritures.

**Résultat attendu :** le Standby applique les modifications du Primary, accepte les lectures et refuse les écritures.

---

# 11. Phase 8 — Intégration finale CI4 / PostgreSQL

## 11.1 Routage des requêtes SQL

- [ ] Vérifier les groupes `primary` et `standby` dans `app/Config/Database.php`.
- [ ] Envoyer les INSERT/UPDATE/DELETE vers `primary`.
- [ ] Envoyer les SELECT compatibles vers `standby` via `ConnexionService::produitsLecture()`.
- [ ] Garder `edit()` et les transactions sur `primary`.
- [ ] Gérer les erreurs de connexion (fallback Standby -> Primary, 503 si Primary down).
- [ ] Tester chaque groupe indépendamment (script PHPUnit ou requête `SELECT 1` via `Database::connect('primary')` / `Database::connect('standby')`).

## 11.2 Vérifications

- [ ] Vérifier que les ajouts/modifications/suppressions arrivent sur le Standby.
- [ ] Vérifier la cohérence des données affichées par CI4.
- [ ] Vérifier le comportement en cas de retard de réplication (bascule lecture sur Primary si besoin).
- [ ] Lancer `composer test` après chaque modification importante.

---

# 12. Phase 9 — Sauvegardes et restauration

**Attention : une réplication n'est pas une sauvegarde indépendante. Une suppression accidentelle sur le Primary peut aussi être répliquée.**

## 12.1 Sauvegarde

- [ ] Mettre en place des sauvegardes avec `pg_dump` ou une stratégie physique.
- [ ] Définir une fréquence, un emplacement distinct du Primary et du Standby, et une conservation.
- [ ] Vérifier les permissions des fichiers.
- [ ] Documenter la procédure dans `documentation/backup.md`.

## 12.2 Restauration

- [ ] Restaurer une sauvegarde sur une instance de test.
- [ ] Vérifier les tables, les données et les contraintes.
- [ ] Documenter le temps de restauration.

## 12.3 Point-in-Time Recovery (PITR)

- [ ] Étudier l'archivage des WAL (`archive_mode`, `archive_command`).
- [ ] Tester une restauration à un instant donné.

---

# 13. Phase 10 — Tests de panne et de résilience

## 13.1 Tests Apache / HAProxy

- [ ] Arrêter Apache 1, vérifier la continuité du service via `/health` et `/produits`.
- [ ] Arrêter Apache 2, vérifier le comportement.
- [ ] Redémarrer les serveurs et vérifier la réintégration des backends.

## 13.2 Tests PostgreSQL

- [ ] Arrêter le Standby : les écritures Primary continuent, les lectures CI4 basculent sur Primary.
- [ ] Redémarrer le Standby et vérifier la reprise de la réplication.
- [ ] Arrêter le Primary : observer les erreurs applicatives (503 attendu).
- [ ] Étudier la promotion du Standby et la reconfiguration de `.env` après promotion.

**Attention :** la réplication seule ne réalise pas un failover automatique. La promotion et le routage après panne nécessitent une stratégie supplémentaire.

## 13.3 Tests de charge

- [ ] Simuler plusieurs requêtes HTTP (HAProxy + `/produits`).
- [ ] Observer la répartition HAProxy et mesurer les temps de réponse.
- [ ] Observer CPU/RAM et les connexions PostgreSQL des deux groupes.
- [ ] Identifier les goulots d'étranglement.

---

# 14. Phase 11 — Sécurité

- [ ] **Retirer le mot de passe en dur de `app/Config/Database.php`** et utiliser `.env`.
- [ ] Ajouter `.env` au `.gitignore` et ne jamais versionner de mot de passe.
- [ ] Créer des utilisateurs PostgreSQL avec des privilèges minimaux (`app_user` ≠ `repl_user`).
- [ ] Restreindre les connexions réseau (`pg_hba.conf`) et configurer le pare-feu.
- [ ] Désactiver `DBDebug` et la toolbar CI4 en production.
- [ ] Protéger le fichier `.env` (droits 640, propriétaire `www-data` ou utilisateur de déploiement).
- [ ] Vérifier les permissions de `writable/` et interdire l'accès HTTP à `app/`, `system/`, `tests/`.
- [ ] Configurer HTTPS si l'application est exposée (et adapter `app.baseURL`).
- [ ] Vérifier les journaux d'accès et d'erreurs (`writable/logs/`).

---

# 15. Phase 12 — Documentation

- [ ] Réaliser un schéma de l'architecture.
- [ ] Documenter les adresses IP et les ports (8080 / 8081 / 8082 / 5432 / 5433).
- [ ] Documenter la configuration Apache, HAProxy et PostgreSQL.
- [ ] Documenter la configuration CI4 (groupes `primary` / `standby`, `.env`, migrations).
- [ ] Documenter les procédures de sauvegarde, restauration et test de panne.
- [ ] Ajouter des captures des résultats.
- [ ] Rédiger le bilan du projet.

---

# 16. Structure finale du dépôt Git

Le dépôt **commence par le projet CI4** ; les dossiers d'infrastructure sont ajoutés à côté, sans déplacer `app/`, `public/` ni `system/`.

```text
CI4/
├── app/                         <- code applicatif CI4 (existant + CRUD à créer)
├── public/                      <- DocumentRoot des Apache
├── writable/                    <- logs, cache, sessions (non versionné)
├── tests/                       <- tests PHPUnit CI4
├── system/                      <- framework CI4 (non modifié)
├── env                          <- modèle de .env
├── composer.json
├── phpunit.xml.dist
├── spark
├── infrastructure/
│   ├── apache/
│   │   ├── serveur1.conf
│   │   └── serveur2.conf
│   ├── haproxy/
│   │   └── haproxy.cfg
│   └── postgresql/
│       ├── primary/
│       ├── standby/
│       └── backup/
├── database/
│   ├── schema.sql
│   └── seed.sql
├── scripts/
│   ├── install.sh
│   ├── backup.sh
│   └── health-check.sh
├── documentation/
│   ├── architecture.md
│   ├── installation.md
│   ├── replication.md
│   └── tests.md
├── .gitignore
└── README.md
```

---

# 17. Critères de réussite

Le projet sera considéré comme fonctionnel lorsque :

- [ ] HAProxy distribue les requêtes entre les deux Apache.
- [ ] L'application reste accessible lorsqu'un Apache tombe.
- [ ] Les deux Apache exécutent la même version du code CI4 (`public/` comme DocumentRoot).
- [ ] Le Primary accepte les lectures et les écritures.
- [ ] Le Standby accepte les lectures et refuse les écritures.
- [ ] Les modifications du Primary sont répliquées vers le Standby.
- [ ] CI4 utilise le groupe `primary` pour les écritures et `standby` pour les lectures réplicables, avec bascule en cas de panne du Standby.
- [ ] Aucun identifiant n'est codé en dur dans `app/Config/Database.php`.
- [ ] Une sauvegarde indépendante peut être restaurée.
- [ ] Les procédures de panne et de reprise sont documentées.
- [ ] `composer test` passe et les tests HTTP/curl valident chaque composant.

---

# 18. Ordre de réalisation recommandé

| Priorité | Module | État | Prochaine action concrète |
|---|---|---|---|
| P0 | Projet CI4 présent | **Fait** | Vérifier `php spark --version` et `composer test` |
| P1 | Préparation Ubuntu + extensions PHP | À faire | Installer `apache2`, `haproxy`, `php-pgsql`, vérifier `php -m` |
| P2 | Apache 1 et Apache 2 (8081/8082) | À faire | VirtualHosts pointant vers `public/` |
| P3 | HAProxy (8080) | À faire | Frontend/backend + `option httpchk GET /health` |
| P4 | `.env` et groupes DB CI4 | **Partiel** | Externaliser les identifiants, ajouter `primary`/`standby` |
| P5 | CRUD CI4 (modèle, contrôleur, vues, migration) | À faire | `php spark make:migration create_produits` |
| P6 | Endpoint `/health` | À faire | Contrôleur `Health` + route |
| P7 | Déploiement sur les deux Apache | À faire | Même commit, `composer install --no-dev`, droits `writable/` |
| P8 | PostgreSQL Primary + réplication | À faire | `wal_level`, `pg_hba.conf`, base `clust_tp1` |
| P9 | Backup initial et Standby (5433) | À faire | `pg_basebackup`, `standby.signal` |
| P10 | Routage SQL dans CI4 | À faire | `ConnexionService`, tests de bascule |
| P11 | Tests de panne | À faire | Arrêter Apache/Standby/Primary et observer |
| P12 | Sécurité et documentation finale | À faire | `.gitignore`, DBDebug, schéma, comptes rendus |

**Principe de travail :** terminer et tester un module avant de passer au suivant (chaque étape doit produire un résultat vérifiable : sortie `curl`, log, test PHPUnit ou capture). Garder le code CI4 inchangé dans `system/` et n'ajouter que du code applicatif dans `app/`.
