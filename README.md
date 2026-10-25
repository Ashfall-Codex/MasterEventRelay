<p align="center">
  <img src="https://repo.ashfall-codex.dev/img/masterevent.png" alt="MasterEvent" width="128" />
</p>

<h1 align="center">MasterEvent Relay</h1>

<p align="center">
  <b>Serveur relais de MasterEvent</b> : synchronisation temps réel des scènes de roleplay entre le Maître du Jeu et ses joueurs.
</p>

<p align="center">
<a href="https://argus.arediss.fr"><img src="https://argus.arediss.fr/api/public/badges/ef24f21c-7f8c-4be4-8974-c802ddea0193/badge.svg" alt="Contrôle qualité" /></a>
<a href="https://argus.arediss.fr"><img src="https://argus.arediss.fr/api/public/badges/ef24f21c-7f8c-4be4-8974-c802ddea0193/badge.svg?metric=reliability" alt="Fiabilité" /></a>
<a href="https://argus.arediss.fr"><img src="https://argus.arediss.fr/api/public/badges/ef24f21c-7f8c-4be4-8974-c802ddea0193/badge.svg?metric=security" alt="Sécurité" /></a>
<a href="https://argus.arediss.fr"><img src="https://argus.arediss.fr/api/public/badges/ef24f21c-7f8c-4be4-8974-c802ddea0193/badge.svg?metric=maintainability" alt="Maintenabilité" /></a>
<a href="https://argus.arediss.fr"><img src="https://argus.arediss.fr/api/public/badges/ef24f21c-7f8c-4be4-8974-c802ddea0193/badge.svg?metric=vulnerabilities" alt="Vulnérabilités" /></a>
</p>

<p align="center">
<a href="https://github.com/Ashfall-Codex/MasterEventRelay/actions/workflows/ci.yml"><img src="https://github.com/Ashfall-Codex/MasterEventRelay/actions/workflows/ci.yml/badge.svg" alt="Contrôle des branches" /></a>
<a href="https://github.com/Ashfall-Codex/MasterEventRelay/actions/workflows/prod.yml"><img src="https://github.com/Ashfall-Codex/MasterEventRelay/actions/workflows/prod.yml/badge.svg" alt="Déploiement" /></a>
<a href="https://github.com/Ashfall-Codex/MasterEventRelay/blob/main/LICENSE"><img src="https://img.shields.io/badge/Licence-GPL--v3-blue.svg" alt="Licence : GPL-v3" /></a>
<a href="https://github.com/Ashfall-Codex/MasterEventRelay/blob/main/AI-DECLARATION.md"><img src="https://img.shields.io/badge/ai_declaration-assist-fef9c3" alt="AI-DECLARATION : assist" /></a>
</p>

Serveur relais du plugin [MasterEvent](https://github.com/Ashfall-Codex/MasterEvent) pour FFXIV : il fait
circuler en temps réel l'état d'une scène de roleplay (marqueurs, PNJ, tours, jets, ambiance) entre le
Maître du Jeu et ses joueurs, et garde ce qui doit survivre à une session (modèles partagés, fiches du
coffre, fichiers des PNJ moddés).

Rust, Axum, Tokio, SQLite. Un seul binaire, sans dépendance système.

## Ce que fait le relais

| Domaine | Rôle |
|---|---|
| Salles (`/` en WebSocket) | une salle par groupe ou par lobby, file d'admission, rôle de MJ et passation, relais des messages |
| Cache de scène | dernier état diffusé par le MJ, rendu à qui revient après une coupure |
| Modèles (`/api/templates`) | partage par code, versions, abonnements |
| Coffre (`/api/cloud/documents`, `/api/account`) | fiches et modèles d'un compte MasterEvent |
| Ashfall Connect (`/api/connect/...`) | façade consommée par Ashfall Connect |
| Fichiers (`/api/files`) | fichiers des PNJ moddés, par salle |
| Santé (`/health`, `/metrics`) | état, nombre de salles actives, version |

Le relais ne lit pas le contenu des scènes : il route, garde le dernier état et contrôle les droits
(seul le MJ ou un joueur promu peut modifier la scène).

## Fonctionnement

- **Axum** + **Tokio** pour les WebSocket et HTTP asynchrones
- **SQLite** (rusqlite) pour le stockage persistant des modèles, avec migration idempotente au démarrage
- Salles par `partyId` ou par code de lobby, expiration après inactivité configurable
- **Couche lobby (protocole 2)** : file d'admission par salle (32 demandes maximum), rattachement des sous-groupes par roster (64 hashes maximum), index `partyId` vers lobby pour la redirection automatique, messages `lobbyPending` / `admit` / `deny` / `lobbyMoved` / `rosterUpdate`
- **Passation de la scène** : le MJ sortant envoie `gmHandover`, le relais ne remet `sceneHandover` (marqueurs avec stats, PNJ) qu'au successeur qui envoie `claimLeadership`, puis annonce `leaderChanged`. Le relais fait foi pour le rôle de MJ (`joinConfirm.leaderHash`)
- **Plancher de version** : `MIN_VERSION` refuse les clients trop anciens, une valeur vide laissant tout passer
- Cache d'état pour récupération de session, jamais persisté sur disque
- **Stockage de modèles** avec codes courts, versioning, statut permanent et hash SHA-256 du créateur
- **Comptes et coffre cloud** (`me_account`, `me_document`) : identifiant public opaque par installation, documents versionnés avec marqueurs de suppression, façade `/api/connect/*` pour Ashfall Connect protégée par secret partagé en comparaison à durée constante
- Nettoyage automatique des rooms (5 min) et modèles expirés (1h)
- **Rate limiting** : 30 messages/s par client connecté, 10 nouvelles connexions/min par IP, 5 créations de salle/h par IP
- Endpoint `/health` (statut, nombre de sessions actives et version du binaire déployé) et `/metrics` (format Prometheus : sessions, clients, uptime, templates, compteurs de messages et d'erreurs)
- Rotation quotidienne des logs avec rétention 7 jours
- **Shutdown gracieux** sur SIGINT / SIGTERM : notification des sessions WS puis drain court avant extinction
- TLS via reverse proxy (Caddy)

## Lancer en local

```bash
cp .env.example .env
cargo run --release
```

Le relais écoute sur `PORT` (8765 par défaut). Pour y brancher le plugin, `RelayServerUrl` dans la
configuration de MasterEvent (`ws://127.0.0.1:8765`), plugin déchargé.

## Configuration

Variables lues dans `.env`, à côté du binaire.

| Variable | Défaut | Rôle |
|---|---|---|
| `PORT`, `HOST` | `8765`, `0.0.0.0` | adresse d'écoute |
| `DATABASE_PATH` | `relay.db` | base SQLite |
| `FILES_DIR` | `relay-files` | fichiers des PNJ moddés |
| `FILE_MAX_BYTES`, `ROOM_FILES_MAX_BYTES` | 200 Mio, 1 Gio | plafonds par fichier et par salle |
| `ROOM_EXPIRY_MS` | 1 h | une salle sans activité depuis ce délai est fermée |
| `TEMPLATE_EXPIRY_MS` | 7 jours | durée de vie d'un modèle partagé non permanent |
| `TOMBSTONE_RETENTION_MS` | 30 jours | conservation des documents supprimés du coffre |
| `MAX_ROOMS` | 1000 | nombre maximal de salles |
| `MIN_VERSION` | vide | version minimale du plugin acceptée |
| `ALLOWED_ORIGINS` | vide | origines acceptées, en HTTP (CORS) comme en WebSocket (en-tête `Origin`) |
| `CONNECT_BASE_URL`, `CONNECT_SERVICE_TOKEN`, `CONNECT_INCOMING_TOKEN` | vides | liaison avec Ashfall Connect |
| `LOG_LEVEL` | `info` | niveau de journalisation |

Deux comportements opposés à connaître :

- `MIN_VERSION` vide **laisse passer** toutes les versions du plugin.
- `CONNECT_SERVICE_TOKEN` ou `CONNECT_INCOMING_TOKEN` vide **ferme** la liaison avec Connect : toute
  requête est refusée.

## Déploiement en production

Pousser du code sur `main` (`src/`, `Cargo.toml`, `Cargo.lock`), c'est déployer : le workflow « Prod »
vérifie le code, compile le binaire x86_64 sous Ubuntu 24.04 et le publie avec son manifeste dans la
release « prod ». Le serveur installe seul chaque nouvelle version, contrôle sa santé et revient à la
précédente en cas d'échec. La documentation ne déclenche aucune publication ; « Run workflow » sur le
workflow Prod en force une.

Comptez 2 à 5 minutes entre le push et le relais à jour. Les changements passent donc par une branche
et une pull request, où tourne la CI ; la fusion dans `main` les met en ligne.

`MIN_VERSION` et les jetons restent dans la configuration du serveur, jamais dans le dépôt.

## Licence

[GPL-3.0](LICENSE), comme le plugin.
