# 🤖 OpenProject AI-Chatbot

Ein KI-gestützter Chatbot, der OpenProject-Kommentare und Work Packages durchsuchbar macht. Nutzer können in natürlicher Sprache Fragen zu Projekten, Aufgaben und Kommentaren stellen und erhalten kontextbezogene Antworten auf Deutsch.

---

## Inhaltsverzeichnis

- [Systemübersicht](#systemübersicht)
- [Voraussetzungen](#voraussetzungen)
- [Installation](#installation)
- [Konfiguration](#konfiguration)
- [n8n-Workflows einrichten](#n8n-workflows-einrichten)
- [Erster Start & Datensynchronisation](#erster-start--datensynchronisation)
- [Nutzung](#nutzung)
- [Architektur](#architektur)
- [Troubleshooting](#troubleshooting)

---

## Systemübersicht

Das System besteht aus folgenden Komponenten, die alle als Docker Container laufen:

| Container     | Beschreibung                                      | Port        |
|---------------|---------------------------------------------------|-------------|
| `open-webui`  | Chat-Oberfläche für den Endnutzer                 | 3000 → 8080 |
| `n8n`         | Workflow-Automatisierung (AI-Agent + Datenpipeline) | 5678        |
| `ollama`      | Lokales LLM (llama3.1:8b) und Embedding-Modell   | 11434       |
| `qdrant`      | Vektor-Datenbank für semantische Suche            | 6333 / 6334 |
| `postgres`    | Chat-Verlauf / Session-Speicher für den AI-Agent  | 5432        |

**Datenfluss:** OpenProject → n8n → Ollama Embeddings → Qdrant  
**Anfrage:** Open WebUI → n8n AI-Agent → Qdrant (semantische Suche) → Ollama LLM → Antwort

---

## Voraussetzungen

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) oder Docker + Docker Compose (v2)
- Mindestens **16 GB RAM** empfohlen (Ollama mit llama3.1:8b benötigt ~8 GB)
- Mindestens **20 GB freier Speicherplatz** (LLM-Modelle)
- Laufende **OpenProject-Instanz** mit API-Zugang
- Betriebssystem: Windows (mit WSL2), macOS oder Linux

---

## Installation

### 1. Repository klonen

```bash
git clone https://github.com/<dein-benutzername>/<repo-name>.git
cd <repo-name>
```

### 2. Umgebungsvariablen konfigurieren

```bash
cp .env.example .env
```

Anschließend die `.env`-Datei mit einem Texteditor öffnen und die Werte eintragen (siehe [Konfiguration](#konfiguration)).

### 3. Container starten

```bash
docker compose up -d
```

Beim ersten Start werden alle Images heruntergeladen – das kann einige Minuten dauern.

### 4. Ollama-Modelle herunterladen

Nach dem ersten Start müssen die KI-Modelle heruntergeladen werden:

```bash
# Sprachmodell (ca. 4,7 GB)
docker exec ollama ollama pull llama3.1:8b

# Embedding-Modell (ca. 270 MB)
docker exec ollama ollama pull nomic-embed-text
```

---

## Konfiguration

Alle Konfigurationswerte werden in der `.env`-Datei gesetzt. Vorlage:

```env
# PostgreSQL
POSTGRES_USER=n8n
POSTGRES_PASSWORD=dein_sicheres_passwort
POSTGRES_DB=n8n

# OpenProject API
# API-Key: OpenProject → Mein Konto → API-Zugriffstoken
OPENPROJECT_API_KEY=dein_openproject_api_key
OPENPROJECT_URL=http://openproject:8080

# n8n
N8N_ENCRYPTION_KEY=ein_zufaelliger_32_zeichen_string
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=dein_sicheres_passwort
```

> **Wichtig:** Die `.env`-Datei niemals in Git einchecken! Sie ist bereits in `.gitignore` eingetragen.

---

## n8n-Workflows einrichten

Die Workflows befinden sich im Ordner `n8n-workflows/`. Sie müssen einmalig in n8n importiert und konfiguriert werden.

### n8n aufrufen

Nach dem Start unter `http://localhost:5678` aufrufen und mit den Zugangsdaten aus der `.env` anmelden.

### Workflows importieren

1. In n8n oben rechts auf **"+"** → **"Import from file"** klicken
2. Folgende Dateien nacheinander importieren:
   - `n8n-workflows/op-api-speichern.json` – Datenpipeline (OpenProject → Qdrant)
   - `n8n-workflows/AI-Agent.json` – Der AI-Chatbot-Agent

### Credentials einrichten

Nach dem Import müssen in n8n die Zugangsdaten (Credentials) eingetragen werden:

| Credential-Name       | Typ              | Werte                                    |
|-----------------------|------------------|------------------------------------------|
| `OpenprojectApi`      | HTTP Basic Auth  | Username: `apikey`, Password: dein API-Key |
| `Ollama account`      | Ollama API       | URL: `http://ollama:11434`               |
| `Qdrant account`      | Qdrant API       | URL: `http://qdrant:6333`                |
| `Postgres account`    | PostgreSQL       | Host: `postgres`, DB/User/PW aus `.env`  |

### Workflows aktivieren

Nach der Credential-Konfiguration beide Workflows über den Toggle oben rechts auf **"Active"** setzen.

---

## Erster Start & Datensynchronisation

Der Workflow `op-api-speichern` synchronisiert automatisch alle 6 Stunden alle OpenProject-Kommentare und Work Packages in die Qdrant-Vektordatenbank.

**Manuelle Erstsynchronisation** (empfohlen nach dem ersten Start):

1. In n8n den Workflow `op-api speichern` öffnen
2. Oben auf **"Test workflow"** klicken
3. Warten bis alle Knoten grün sind (je nach Datenmenge mehrere Minuten)

---

## Nutzung

### Chat-Oberfläche (Open WebUI)

`http://localhost:3000` im Browser öffnen.

Der Chatbot beantwortet Fragen zu OpenProject auf Deutsch, zum Beispiel:

- *„Welche Work Packages sind im Projekt X offen?"*
- *„Was hat Mitarbeiter Y zuletzt kommentiert?"*
- *„Zeig mir alle Kommentare zu Work Package #42."*
- *„Welche Aufgaben sind noch nicht zugewiesen?"*

### Direkte API-Nutzung

Der AI-Agent ist auch per Webhook erreichbar (z.B. für Integration in andere Tools):

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
┌─────────────────────────────────────────────────────┐
│                   Datenpipeline (alle 6h)            │
│  OpenProject API → Work Packages & Kommentare        │
│       → Ollama (nomic-embed-text Embeddings)         │
│       → Qdrant (Vektordatenbank)                     │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│                   Chat-Anfrage                       │
│  Open WebUI / Webhook                                │
│       → n8n AI-Agent                                 │
│       → Qdrant (Top-200 semantisch ähnliche Einträge)│
│       → Ollama LLM (llama3.1:8b)                    │
│       → Antwort auf Deutsch                          │
│  Postgres speichert den Chat-Verlauf (Session)       │
└─────────────────────────────────────────────────────┘
```

---

## Troubleshooting

**Container startet nicht**
```bash
docker compose logs <container-name>
# z.B.: docker compose logs n8n
```

**Ollama-Modell fehlt**
```bash
docker exec ollama ollama list
# Wenn leer: Modelle erneut pullen (siehe Schritt 4 der Installation)
```

**Qdrant-Datenbank ist leer / Chatbot weiß nichts**  
→ Datenpipeline in n8n manuell ausführen (siehe [Erster Start](#erster-start--datensynchronisation))

**n8n Credentials funktionieren nicht**  
→ Sicherstellen, dass die Container-Namen in den URLs stimmen (z.B. `http://ollama:11434`, nicht `localhost`). Die Container kommunizieren über das interne Docker-Netzwerk.

**Open WebUI nicht erreichbar**  
→ `docker ps` prüfen ob alle Container laufen. Beim ersten Start kann Open WebUI 1–2 Minuten brauchen.

---

## Lizenz

Dieses Projekt ist ein Praxisprojekt und nicht für den öffentlichen Einsatz vorgesehen.
