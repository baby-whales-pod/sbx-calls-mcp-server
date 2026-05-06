# Connecter une sandbox sbx à la MCP Gateway

Ce document explique comment permettre à une sandbox `sbx` (Docker Sandboxes) de communiquer avec la MCP Gateway lancée via `docker compose -f mcp-scripts/compose.dev.yml up`.

---

## Architecture

```
sandbox (ex: claude)
  └── http://host.docker.internal:9011/mcp
        └── mcp-gateway  (port 9011, exposé sur l'hôte)
              └── http://mcp-scripts:8080/mcp  (réseau Docker interne)
                    └── mcp-scripts  (serveur MCP Go)
```

Le compose démarre trois services :

| Service        | Port hôte | Description                              |
|----------------|-----------|------------------------------------------|
| `mcp-scripts`  | 8080      | Serveur MCP Go (accès direct)            |
| `mcp-gateway`  | 9011      | Gateway Docker MCP (accès sécurisé)      |
| `mcp-inspector`| 6274      | Interface web de test manuel             |

---

## Pourquoi `localhost` ne fonctionne pas dans la sandbox

Une sandbox sbx possède son propre réseau isolé. À l'intérieur, `localhost` désigne la sandbox elle-même, pas la machine hôte. Pour atteindre un service qui tourne sur l'hôte, il faut utiliser l'adresse spéciale :

```
host.docker.internal
```

De plus, le proxy réseau de la sandbox bloque par défaut toutes les connexions sortantes non déclarées. Il faut donc explicitement autoriser les ports cibles.

---

## Étape 1 — Autoriser les ports dans la policy réseau

Sur la **machine hôte**, autoriser la sandbox à atteindre le port de la gateway :

```bash
# Via la gateway (recommandé)
sbx policy allow network localhost:9011

# Ou en accès direct au serveur MCP
sbx policy allow network localhost:8080
```

Pour vérifier les règles actives :

```bash
sbx policy ls
sbx policy log
```

---

## Étape 2 — Configurer le client MCP dans la sandbox

### Option A — Configuration manuelle (sandbox déjà démarrée)

Se connecter à la sandbox et exécuter :

```bash
claude mcp add mcp-scripts --transport http http://host.docker.internal:9011/mcp
```

Vérifier que le serveur est bien enregistré :

```bash
claude mcp list
```

### Option B — Configuration permanente dans le spec de l'agent

Modifier les deux fichiers suivants dans le projet `sandboxes` pour que chaque nouvelle sandbox soit pré-configurée.

#### `sandboxes/sandboxlib/kit/agents/claude/spec.yaml`

Ajouter `host.docker.internal` dans la liste des domaines autorisés :

```yaml
network:
  serviceDomains:
    api.anthropic.com: anthropic
    console.anthropic.com: anthropic
    claude.ai: anthropic
  serviceAuth:
    anthropic:
      headerName: x-api-key
      valueFormat: "%s"
  allowedDomains:
    - "downloads.claude.ai:443"
    - "claude.com:443"
    - "host.docker.internal:9011"   # gateway MCP
    - "host.docker.internal:8080"   # accès direct (optionnel)
```

#### `sandboxes/sandboxlib/kit/agents/claude/files/home/.claude.json`

Pré-configurer le serveur MCP dans le fichier de config Claude Code :

```json
{
    "bypassPermissionsModeAccepted": true,
    "hasCompletedOnboarding": true,
    "mcpServers": {
        "mcp-scripts": {
            "type": "http",
            "url": "http://host.docker.internal:9011/mcp"
        }
    }
}
```

---

## Étape 3 — Vérifier la connexion

Depuis l'intérieur de la sandbox, tester que la gateway répond :

```bash
curl http://host.docker.internal:9011/mcp
```

Ou via Claude Code, lister les outils MCP disponibles :

```bash
claude mcp list
```

---

## Démarrage du stack MCP

Depuis le répertoire `mcp-scripts` :

```bash
# Premier démarrage (build de l'image)
docker compose -f compose.dev.yml up --build -d

# Démarrages suivants
docker compose -f compose.dev.yml up -d

# Suivre les logs
docker compose -f compose.dev.yml logs -f

# Arrêter
docker compose -f compose.dev.yml down
```

L'inspector est accessible à l'adresse : [http://localhost:6274](http://localhost:6274)

---

## Résumé des prérequis

| Prérequis | Commande |
|-----------|----------|
| Stack MCP démarré | `docker compose -f mcp-scripts/compose.dev.yml up -d` |
| Port gateway autorisé | `sbx policy allow network localhost:9011` (sur l'hôte) |
| Client MCP configuré | `claude mcp add mcp-scripts --transport http http://host.docker.internal:9011/mcp` (dans la sandbox) |
