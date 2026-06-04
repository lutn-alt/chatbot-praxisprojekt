# 🤖 OpenProject AI-Chatbot

Ein KI-gestützter Chatbot, der OpenProject-Kommentare und Work Packages durchsuchbar macht. Nutzer können in natürlicher Sprache Fragen zu Projekten, Aufgaben und Kommentaren stellen und erhalten kontextbezogene Antworten auf Deutsch.

---

## Inhaltsverzeichnis

- [Repo-Struktur](#repo-struktur)
- [Systemübersicht](#systemübersicht)
- [Voraussetzungen](#voraussetzungen)
- [Installation](#installation)
- [Docker Images herunterladen](#docker-images-herunterladen)
- [Ollama-Modelle herunterladen](#ollama-modelle-herunterladen)
- [n8n-Workflows einrichten](#n8n-workflows-einrichten)
- [Erster Start & Datensynchronisation](#erster-start--datensynchronisation)
- [Nutzung](#nutzung)
- [Architektur](#architektur)
- [Troubleshooting](#troubleshooting)

---

## Repo-Struktur

```
chatbot-praxisprojekt/
├── README.md                   ← diese Datei
├── docker-compose.yml          ← startet alle Services
├── n8n_workflows.zip           ← n8n-Workflows (ZIP-Archiv)
│   ├── AI-Agent.json           ← Chatbot-Agent
│   └── op-api-speichern.json   ← Datenpipeline OpenProject → Qdrant
└── docs/                       ← Dokumentation (folgt)
```

> Die Workflows sind als ZIP-Archiv beigefügt. Vor dem Import in n8n einfach entpacken.

---

## Systemübersicht

Das System besteht aus folgenden Komponenten, die alle als Docker Container laufen:

| Container     | Image                                  | Beschreibung                                        | Port         |
|---------------|----------------------------------------|-----------------------------------------------------|--------------|
| `qdrant`      | `qdrant/qdrant:latest`                 | Vektor-Datenbank für semantische Suche              | 6333 / 6334  |
| `postgres`    | `postgres:16`                          | Datenbank für n8n und Chat-Verlauf                  | 5432         |
| `ollama`      | `ollama/ollama:latest`                 | Lokales LLM-Backend (llama3.1:8b + nomic-embed-text)| 11434        |
| `open-webui`  | `ghcr.io/open-webui/open-webui:main`   | Chat-Oberfläche für den Endnutzer                   | 3000 → 8080  |
| `n8n`         | `docker.n8n.io/n8nio/n8n:latest`       | Workflow-Automatisierung (AI-Agent + Datenpipeline) | 5678         |

**Datenfluss:** OpenProject → n8n → Ollama Embeddings → Qdrant  
**Anfrage:** Open WebUI → n8n AI-Agent → Qdrant (semantische Suche) → Ollama LLM → Antwort

---

## Voraussetzungen

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) oder Docker + Docker Compose (v2)
- Mindestens **16 GB RAM** empfohlen (Ollama mit llama3.1:8b benötigt ~8 GB)
- Mindestens **20 GB freier Speicherplatz** (LLM-Modelle + Container-Images)
- Laufende **OpenProject-Instanz** mit API-Zugang
- Betriebssystem: Windows (mit WSL2), macOS oder Linux

---

## Installation

### 1. Repository klonen

```bash
git clone https://github.com/<dein-benutzername>/<repo-name>.git
cd <repo-name>
```

### 2. Passwörter in der docker-compose.yml anpassen

Vor dem ersten Start die drei `changeme`-Stellen in der `docker-compose.yml` durch sichere Passwörter ersetzen:

```yaml
# postgres-Service:
POSTGRES_PASSWORD: dein_sicheres_passwort

# n8n-Service:
DB_POSTGRESDB_PASSWORD: dein_sicheres_passwort   # gleich wie oben!
N8N_BASIC_AUTH_PASSWORD: dein_sicheres_passwort
```

> **Wichtig:** Alle drei Passwörter für `POSTGRES_PASSWORD` und `DB_POSTGRESDB_PASSWORD` müssen identisch sein, da n8n dieselbe Postgres-Datenbank nutzt.

### 3. Container starten

```bash
docker compose up -d
```

Beim ersten Start werden alle Images automatisch aus dem Internet gezogen – das kann je nach Internetgeschwindigkeit **5–15 Minuten** dauern (Gesamtgröße ca. 10–15 GB).

---

## Docker Images herunterladen

Die Images werden beim ersten `docker compose up -d` **automatisch** gezogen. Falls du sie manuell vorab herunterladen möchtest (z.B. für Offline-Installation), können die einzelnen Images so gezogen werden:

```bash
# Vektor-Datenbank
docker pull qdrant/qdrant:latest

# Relationale Datenbank
docker pull postgres:16

# LLM-Backend
docker pull ollama/ollama:latest

# Chat-Oberfläche
docker pull ghcr.io/open-webui/open-webui:main

# Workflow-Automatisierung
docker pull docker.n8n.io/n8nio/n8n:latest
```

Status aller laufenden Container prüfen:

```bash
docker compose ps
```

---

## Ollama-Modelle herunterladen

Nach dem ersten Start von Ollama müssen die KI-Modelle einmalig heruntergeladen werden. Diese werden **dauerhaft im Docker Volume** gespeichert und müssen nicht erneut gezogen werden:

```bash
# Sprachmodell für den Chatbot (ca. 4,7 GB)
docker exec ollama ollama pull llama3.1:8b

# Embedding-Modell für die Vektorsuche (ca. 270 MB)
docker exec ollama ollama pull nomic-embed-text
```

Installierte Modelle anzeigen:

```bash
docker exec ollama ollama list
```

---

## n8n-Workflows einrichten

### n8n aufrufen

`http://localhost:5678` im Browser aufrufen und mit den Zugangsdaten aus der `docker-compose.yml` anmelden (`admin` / das gesetzte Passwort).

### Workflows importieren

1. Das Archiv `n8n_workflows.zip` entpacken
2. In n8n oben links auf **"Workflows"** → oben rechts **"..."** → **"Import from file"** klicken
3. Folgende Dateien nacheinander importieren:
   - `op-api-speichern.json` – Datenpipeline (OpenProject → Qdrant, läuft alle 6h)
   - `AI-Agent.json` – Der eigentliche Chatbot-Agent

### Credentials einrichten

Nach dem Import der Workflows müssen in n8n einmalig die Zugangsdaten hinterlegt werden. Dies geschieht unter **"Settings"** → **"Credentials"** → **"Add Credential"**.

Folgende Credentials werden benötigt:

---

#### 1. Ollama account (Typ: `Ollama API`)

Wird von: `AI-Agent`, `op-api-speichern`

| Feld    | Wert                    |
|---------|-------------------------|
| Base URL | `http://ollama:11434`  |

> Wichtig: Nicht `localhost`, sondern den Container-Namen `ollama` verwenden – die Container kommunizieren intern über das Docker-Netzwerk.

---

#### 2. Qdrant account (Typ: `Qdrant API`)

Wird von: `AI-Agent`

| Feld    | Wert                   |
|---------|------------------------|
| API URL | `http://qdrant:6333`   |
| API Key | *(leer lassen)*        |

---

#### 3. Postgres account (Typ: `Postgres`)

Wird von: `AI-Agent` (Chat-Memory)

| Feld      | Wert                                        |
|-----------|---------------------------------------------|
| Host      | `postgres`                                  |
| Port      | `5432`                                      |
| Database  | `n8n`                                       |
| User      | `admin`                                     |
| Password  | *(das in docker-compose.yml gesetzte PW)*   |
| SSL       | `disabled`                                  |

---

#### 4. OpenprojectApi (Typ: `HTTP Basic Auth`)

Wird von: `op-api-speichern`

| Feld     | Wert                                                                 |
|----------|----------------------------------------------------------------------|
| User     | `apikey`                                                             |
| Password | Dein OpenProject API-Key *(OpenProject → Mein Konto → API-Token)*   |

---

### Credentials den Workflow-Knoten zuweisen

Nach dem Anlegen der Credentials müssen diese in den importierten Workflows den jeweiligen Knoten zugewiesen werden:

1. Workflow öffnen
2. Jeden rot markierten Knoten anklicken (rot = fehlende Credential)
3. Im Dropdown die passende Credential auswählen
4. Speichern

### Workflows aktivieren

Beide Workflows über den Toggle oben rechts auf **"Active"** setzen.

---

## Erster Start & Datensynchronisation

Der Workflow `op-api-speichern` synchronisiert automatisch alle **6 Stunden** alle OpenProject-Kommentare und Work Packages in die Qdrant-Vektordatenbank.

**Manuelle Erstsynchronisation** – direkt nach dem Setup empfohlen, damit der Chatbot sofort Daten hat:

1. In n8n den Workflow `op-api speichern` öffnen
2. Oben auf **"Test workflow"** klicken
3. Warten bis alle Knoten grün sind (je nach Datenmenge 1–10 Minuten)

---

## Nutzung

### Chat-Oberfläche (Open WebUI)

`http://localhost:3000` im Browser öffnen.

Der Chatbot beantwortet Fragen zu OpenProject auf Deutsch, zum Beispiel:

- *„Welche Work Packages sind im Projekt X offen?"*
- *„Was hat Mitarbeiter Y zuletzt kommentiert?"*
- *„Zeig mir alle Kommentare zu Work Package #42."*
- *„Welche Aufgaben sind noch nicht zugewiesen?"*

### Direkte API-Nutzung (Webhook)

Der AI-Agent ist auch per HTTP erreichbar, z.B. für die Integration in andere Tools:

```bash
curl -X POST http://localhost:5678/webhook/invoke_n8n_agent \
  -H "Content-Type: application/json" \
  -d '{
    "sessionId": "session-123",
    "chatInput": "Welche Work Packages sind offen?"
  }'
```

---

## Architektur

```
┌─────────────────────────────────────────────────────────┐
│              Datenpipeline  (alle 6 Stunden)             │
│                                                          │
│  OpenProject API                                         │
│    → Alle Projekte, Work Packages, Kommentare            │
│    → Ollama  (nomic-embed-text  →  Vektoren)             │
│    → Qdrant  (Vektordatenbank speichert Einträge)        │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                     Chat-Anfrage                         │
│                                                          │
│  Nutzer tippt Frage in Open WebUI (Port 3000)            │
│    → n8n AI-Agent                                        │
│    → Qdrant  (Top-200 semantisch ähnliche Einträge)      │
│    → Ollama LLM  (llama3.1:8b  →  Antwort auf Deutsch)  │
│  Postgres speichert den Chat-Verlauf je Session          │
└─────────────────────────────────────────────────────────┘
```

---

## Troubleshooting

**Container startet nicht**
```bash
docker compose logs <container-name>
# z.B.: docker compose logs n8n
```

**Ollama-Modell fehlt / Chatbot antwortet nicht**
```bash
docker exec ollama ollama list
# Wenn leer → Modelle erneut pullen (siehe "Ollama-Modelle herunterladen")
```

**Qdrant-Datenbank ist leer / Chatbot weiß nichts über OpenProject**  
→ Datenpipeline in n8n manuell ausführen (siehe [Erster Start](#erster-start--datensynchronisation))

**n8n Credentials funktionieren nicht**  
→ Sicherstellen, dass in den URLs die **Container-Namen** stehen (z.B. `http://ollama:11434`), nicht `localhost`. Container kommunizieren intern über das Docker-Netzwerk.

**Knoten in n8n rot markiert nach Import**  
→ Credential noch nicht zugewiesen. Knoten anklicken und passende Credential im Dropdown auswählen.

**Open WebUI nicht erreichbar**  
→ `docker compose ps` prüfen ob alle Container den Status `Up` haben. Beim ersten Start kann Open WebUI 1–2 Minuten brauchen.

**Postgres-Verbindung schlägt fehl**  
→ Prüfen ob `POSTGRES_PASSWORD` in `docker-compose.yml` und `DB_POSTGRESDB_PASSWORD` identisch sind.

---

## Dokumentation

Weiterführende Dokumentation wird im Ordner `docs/` abgelegt (folgt).

---

## Lizenz

Dieses Projekt ist ein Praxisprojekt und nicht für den öffentlichen Einsatz vorgesehen.
