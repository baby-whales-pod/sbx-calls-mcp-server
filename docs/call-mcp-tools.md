# Appeler les outils MCP

Ce document explique comment appeler les outils exposés par le serveur MCP, une fois le stack démarré (`docker compose -f mcp-scripts/compose.dev.yml up`).

---

## Outils disponibles

| Outil         | Arguments requis          | Arguments optionnels              | Description                          |
|---------------|---------------------------|-----------------------------------|--------------------------------------|
| `get-date`    | —                         | `format` (string)                 | Date/heure courante (ISO 8601)       |
| `system-info` | —                         | —                                 | OS, kernel, CPU, mémoire             |
| `list-files`  | `path` (string)           | `show_hidden` (boolean)           | Liste les fichiers d'un répertoire   |
| `greet`       | `name` (string)           | `language` (string : en, fr, es)  | Salutation personnalisée             |
| `echo`        | `message` (string)        | `upper` (boolean)                 | Répète le message                    |
| `add`         | `a` (number), `b` (number)| —                                 | Somme de deux nombres                |
| `word-count`  | `text` (string)           | —                                 | Compte caractères, mots et lignes    |

---

## Méthode 1 — Via Claude Code (dans la sandbox)

Une fois le serveur MCP enregistré dans Claude Code, les outils sont disponibles automatiquement. Claude les utilise de lui-même quand la demande s'y prête.

**Usage naturel :**
> "Liste les fichiers du répertoire /app"
> "Quelle heure est-il ?"
> "Calcule 42 + 58"

**Appel explicite :**
> "Utilise l'outil `get-date` pour me donner la date au format `+%d/%m/%Y`"

Pour vérifier que le serveur MCP est bien enregistré :

```bash
claude mcp list
```

---

## Méthode 2 — Via curl (protocole MCP Streamable HTTP)

Le transport utilisé est **Streamable HTTP** : chaque appel est un POST JSON-RPC 2.0 sur l'endpoint `/mcp`.

> Depuis la **machine hôte**, utiliser `http://localhost:9011/mcp`.  
> Depuis l'**intérieur d'une sandbox**, utiliser `http://host.docker.internal:9011/mcp`.

### Initialisation de session (obligatoire)

Le protocole MCP Streamable HTTP requiert une initialisation avant tout appel d'outil. Il faut :

1. Envoyer `initialize` et récupérer le `mcp-session-id` dans les headers de réponse
2. Confirmer avec `notifications/initialized`
3. Appeler les outils en passant le session ID dans chaque requête

Le script suivant encapsule ces trois étapes :

```bash
# 1. Initialiser la session et récupérer le session ID
SESSION_ID=$(curl -s -D - -X POST http://localhost:9011/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{
    "jsonrpc": "2.0",
    "method": "initialize",
    "params": {
      "protocolVersion": "2024-11-05",
      "capabilities": {},
      "clientInfo": { "name": "curl-client", "version": "1.0" }
    },
    "id": 1
  }' | grep -i "mcp-session-id" | awk '{print $2}' | tr -d '\r')

echo "Session ID: $SESSION_ID"

# 2. Confirmer l'initialisation
curl -s -X POST http://localhost:9011/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "mcp-session-id: $SESSION_ID" \
  -d '{"jsonrpc":"2.0","method":"notifications/initialized"}' > /dev/null
```

Le `SESSION_ID` est ensuite passé via le header `mcp-session-id` dans tous les appels suivants.

### Lister les outils disponibles

```bash
curl -s -X POST http://localhost:9011/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "mcp-session-id: $SESSION_ID" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/list",
    "id": 1
  }'
```

### Appeler `get-date`

Sans argument :

```bash
curl -s -X POST http://localhost:9011/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "mcp-session-id: $SESSION_ID" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/call",
    "params": {
      "name": "get-date",
      "arguments": {}
    },
    "id": 2
  }'
```

Avec un format personnalisé :

```bash
curl -s -X POST http://localhost:9011/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "mcp-session-id: $SESSION_ID" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/call",
    "params": {
      "name": "get-date",
      "arguments": { "format": "+%d/%m/%Y" }
    },
    "id": 2
  }'
```

### Appeler `system-info`

```bash
curl -s -X POST http://localhost:9011/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "mcp-session-id: $SESSION_ID" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/call",
    "params": {
      "name": "system-info",
      "arguments": {}
    },
    "id": 2
  }'
```

### Appeler `list-files`

```bash
curl -s -X POST http://localhost:9011/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "mcp-session-id: $SESSION_ID" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/call",
    "params": {
      "name": "list-files",
      "arguments": { "path": "/tmp", "show_hidden": false }
    },
    "id": 2
  }'
```

### Appeler `greet`

```bash
curl -s -X POST http://localhost:9011/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "mcp-session-id: $SESSION_ID" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/call",
    "params": {
      "name": "greet",
      "arguments": { "name": "Alice", "language": "fr" }
    },
    "id": 2
  }'
```

### Appeler `echo`

```bash
curl -s -X POST http://localhost:9011/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "mcp-session-id: $SESSION_ID" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/call",
    "params": {
      "name": "echo",
      "arguments": { "message": "hello world", "upper": true }
    },
    "id": 2
  }'
```

### Appeler `add`

```bash
curl -s -X POST http://localhost:9011/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "mcp-session-id: $SESSION_ID" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/call",
    "params": {
      "name": "add",
      "arguments": { "a": 42, "b": 58 }
    },
    "id": 2
  }'
```

### Appeler `word-count`

```bash
curl -s -X POST http://localhost:9011/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "mcp-session-id: $SESSION_ID" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/call",
    "params": {
      "name": "word-count",
      "arguments": { "text": "Bonjour le monde, ceci est un test." }
    },
    "id": 2
  }'
```

---

## Méthode 3 — Via MCP Inspector (interface graphique)

Ouvrir **[http://localhost:6274](http://localhost:6274)** dans un navigateur.

Le serveur `mcp-scripts` est pré-configuré en deux entrées :

| Entrée                    | URL                              | Description                    |
|---------------------------|----------------------------------|--------------------------------|
| `mcp-scripts (via gateway)` | `http://localhost:9011/mcp`    | Accès sécurisé via la gateway  |
| `mcp-scripts (direct)`    | `http://localhost:8080/mcp`      | Accès direct au serveur MCP    |

L'inspector permet de :
- Parcourir la liste des outils avec leur documentation
- Remplir les arguments via un formulaire
- Visualiser les réponses JSON brutes

---

## Format d'une réponse JSON-RPC

Une réponse type pour un appel d'outil :

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "2026-05-06T14:32:00Z"
      }
    ]
  }
}
```

En cas d'erreur :

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "error": {
    "code": -32602,
    "message": "argument 'path' is required"
  }
}
```
