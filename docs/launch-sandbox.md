# Lancer une sandbox connectée à la MCP Gateway

Ce document décrit les étapes pour démarrer le stack MCP sur la machine hôte et créer une sandbox Claude préconfigurée pour s'y connecter.

---

## Prérequis

- `sbx` CLI installé sur la machine hôte
- Docker et Docker Compose disponibles
- Être dans le répertoire `sbx-calls-mcp-server`

---

## Architecture

```
sandbox Claude
  └── claude mcp (mcp-scripts) → http://host.docker.internal:9011/mcp
        └── mcp-gateway  (port 9011, exposé sur l'hôte)
              └── mcp-scripts (port 8080, réseau Docker interne)
```

| Service         | Port hôte | Description                          |
|-----------------|-----------|--------------------------------------|
| `mcp-scripts`   | 8080      | Serveur MCP Go (accès direct)        |
| `mcp-gateway`   | 9011      | Gateway Docker MCP (accès sécurisé)  |
| `mcp-inspector` | 6274      | Interface web de test manuel         |

---

## Étape 1 — Autoriser le port de la gateway

Sur la **machine hôte**, autoriser la sandbox à joindre la gateway :

```bash
sbx policy allow network localhost:9011
```

> Cette règle persiste entre les sessions. À n'exécuter qu'une seule fois.

---

## Étape 2 — Démarrer le stack MCP

```bash
# Premier démarrage : build de l'image mcp-scripts depuis les sources
docker compose -f mcp-scripts/compose.dev.yml up --build -d

# Démarrages suivants (image déjà buildée)
docker compose -f mcp-scripts/compose.dev.yml up -d
```

Vérifier que les trois services sont `healthy` / `running` :

```bash
docker compose -f mcp-scripts/compose.dev.yml ps
```

Suivre les logs si besoin :

```bash
docker compose -f mcp-scripts/compose.dev.yml logs -f
```

---

## Étape 3 — Créer la sandbox avec le kit

```bash
sbx create claude --kit ./kit .
```

> Le kit `./kit` :
> - autorise les connexions vers `host.docker.internal:9011` dans la policy réseau
> - exécute `claude mcp add` au démarrage pour enregistrer la gateway dans Claude Code

Lancer la sandbox :

```bash
sbx run claude-sbx-calls-mcp-server
```

---

## Étape 4 — Vérifier la connexion MCP dans la sandbox

Une fois dans la sandbox, vérifier que le serveur MCP est bien enregistré :

```bash
claude mcp list
```

Résultat attendu :

```
mcp-scripts: http://host.docker.internal:9011/mcp (http)
```

Tester la connectivité réseau vers la gateway :

```bash
curl http://host.docker.internal:9011/mcp
```

---

## Injecter le kit dans une sandbox existante

Si la sandbox est déjà créée sans le kit :

```bash
sbx kit add <nom-sandbox> ./kit
```

Puis redémarrer la sandbox pour que le `startup` command s'exécute et enregistre le serveur MCP.

---

## Arrêter le stack MCP

```bash
docker compose -f mcp-scripts/compose.dev.yml down
```

---

## MCP Inspector (optionnel)

L'inspector permet de parcourir et d'appeler les outils MCP manuellement depuis un navigateur :

```
http://localhost:6274
```

Deux connexions sont préconfigurées :
- `mcp-scripts (via gateway)` — accès sécurisé via la gateway
- `mcp-scripts (direct)` — accès direct au serveur sur le port 8080

---

## Résumé des commandes

```bash
# 1. Autoriser le port (une seule fois)
sbx policy allow network localhost:9011

# 2. Démarrer le stack MCP
docker compose -f mcp-scripts/compose.dev.yml up --build -d

# 3. Créer et lancer la sandbox
sbx create claude --kit ./kit .
sbx run claude-sbx-calls-mcp-server

# 4. Vérifier (depuis la sandbox)
claude mcp list
```
