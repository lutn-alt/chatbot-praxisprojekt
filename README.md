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
- [Open WebUI – N8N Pipe einrichten](#open-webui--n8n-pipe-einrichten)
- [Erster Start & Datensynchronisation](#erster-start--datensynchronisation)
- [Nutzung](#nutzung)
- [Architektur](#architektur)
- [Troubleshooting](#troubleshooting)
- [Möglichkeiten der Erweiterung](#möglichkeiten-der-erweiterung)

---

## Repo-Struktur

```
chatbot-praxisprojekt/
├── README.md                   ← diese Datei
├── docker-compose.yml          ← startet alle Services
├── n8n_workflows.zip           ← n8n-Workflows (ZIP-Archiv)
│   ├── AI-Agent.json           ← Chatbot-Agent
│   └── op-api-speichern.json   ← Datenpipeline OpenProject → Qdrant
├── n8n_pipe.json               ← Open WebUI Pipe-Funktion (verbindet WebUI ↔ n8n)
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

**Datenfluss:** OpenProject → n8n (Sync alle 6h) → Ollama Embeddings → Qdrant  
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

## Open WebUI – N8N Pipe einrichten

Die **N8N Pipe** ist eine Funktion für Open WebUI, die die Chat-Oberfläche direkt mit dem n8n AI-Agent verbindet. Ohne sie würde Open WebUI nur lokal installierte Ollama-Modelle nutzen, mit der Pipe werden alle Nachrichten stattdessen an den n8n-Workflow weitergeleitet, der Qdrant-Suche, Chat-Memory und die OpenProject-Daten einbindet.

Quelle / Original: [N8N Pipe auf openwebui.com](https://openwebui.com/posts/c82c9b29-c517-4deb-bd42-d058aa889633)

### Schritt 1: Funktion importieren

1. Open WebUI aufrufen: `http://localhost:3000`
2. Als Admin anmelden
3. Oben im Menü auf **"Functions"** klicken
4. Oben rechts auf **"Import"** klicken
5. Die Datei `n8n_pipe.json` aus dem Repository auswählen und importieren
6. Die Funktion erscheint anschließend als **„N8N Pipe v0.2.0"** in der Liste
7. Den Toggle rechts neben der Funktion auf **aktiv** (grün) setzen

### Schritt 2: Valves konfigurieren

Die Pipe muss mit der URL des n8n-Webhooks und einem optionalen Bearer Token konfiguriert werden. Dazu auf das **Zahnrad-Symbol** neben der N8N Pipe klicken – es öffnet sich das Valves-Fenster:

| Feld                    | Wert                                          | Beschreibung                                      |
|-------------------------|-----------------------------------------------|---------------------------------------------------|
| **N8N Url**             | `http://n8n:5678/webhook/invoke_n8n_agent`    | Webhook-URL des AI-Agent-Workflows in n8n         |
| **N8N Bearer Token**    | `...` *(leer lassen oder eigenen Token setzen)* | Absicherung des Webhooks – in dieser Installation nicht aktiv |
| **Input Field**         | `chatInput`                                   | Feldname für die Nutzernachricht (nicht ändern)   |
| **Response Field**      | `output`                                      | Feldname der Antwort aus n8n (nicht ändern)       |
| **Emit Interval**       | `2`                                           | Sekunden zwischen Status-Updates                  |
| **Enable Status Indicator** | aktiv                                    | Zeigt „Calling N8N Workflow…" während der Antwort |

Anschließend auf **"Save"** klicken.

> **Hinweis zur URL:** Da Open WebUI und n8n im selben Docker-Netzwerk laufen, wird der Container-Name `n8n` als Hostname verwendet, nicht `localhost`.

### Schritt 3: Pipe als Modell auswählen

Beim Starten eines neuen Chats in Open WebUI im Modell-Dropdown oben **„N8N Pipe"** auswählen. Ab jetzt werden alle Nachrichten über n8n bearbeitet und mit den OpenProject-Daten beantwortet.

---

## Erster Start & Datensynchronisation

Der Workflow `op-api-speichern` synchronisiert automatisch alle OpenProject-Kommentare und Work Packages in die Qdrant-Vektordatenbank.

**Manuelle Erstsynchronisation**, direkt nach dem Setup empfohlen, damit der Chatbot sofort Daten hat:

1. In n8n den Workflow `op-api speichern` öffnen
2. Oben auf **"Test workflow"** klicken
3. Warten bis alle Knoten grün sind (je nach Datenmenge variiert)

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
│                    Datenpipeline                         │
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
│    → Ollama LLM  (llama3.1:8b  →  Antwort auf Deutsch)   │
│  Postgres speichert den Chat-Verlauf je Session          │
└─────────────────────────────────────────────────────────┘
```

---

## Troubleshooting

**N8N Pipe antwortet nicht / Fehler im Chat**  
→ Sicherstellen, dass die N8N Url in den Valves `http://n8n:5678/webhook/invoke_n8n_agent` lautet (Container-Name, nicht `localhost`). Den AI-Agent-Workflow in n8n auf „Active" prüfen.

**N8N Pipe taucht im Modell-Dropdown nicht auf**  
→ In Open WebUI unter **Functions** prüfen ob der Toggle der N8N Pipe grün (aktiv) ist.

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

---

## Möglichkeiten der Erweiterung

### GPU-Anbindung (dringend empfohlen)

> ⚠️ **Wichtiger Hinweis:** Ohne GPU läuft das LLM (`llama3.1:8b`) ausschließlich auf der CPU. Die Antwortzeiten betragen dann **wesentlich länger** und machen den Chatbot für den produktiven Einsatz praktisch unbrauchbar. Eine NVIDIA-GPU verbessert die Performance, das ist kein optionales Upgrade, sondern eine Grundvoraussetzung für sinnvolle Nutzung.

#### Voraussetzungen

- NVIDIA-Grafikkarte (mindestens 8 GB VRAM für llama3.1:8b)
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html) installiert
- Aktueller NVIDIA-Treiber

#### Aktivierung in der docker-compose.yml

Den auskommentierten GPU-Block im `ollama`-Service einkommentieren:

```yaml
ollama:
  image: ollama/ollama:latest
  # ...
  deploy:
    resources:
      reservations:
        devices:
          - driver: nvidia
            count: all
            capabilities: [gpu]
```

Danach Container neu starten:

```bash
docker compose down
docker compose up -d
```

GPU-Nutzung prüfen:

```bash
docker exec ollama ollama ps
# oder auf dem Host:
nvidia-smi
```

---

### Weitere Systeme anbinden

Das System ist modular aufgebaut, weitere Datenquellen lassen sich über neue n8n-Workflows anbinden. Die Daten werden dabei immer nach demselben Prinzip verarbeitet: Inhalte abrufen → Embeddings erzeugen (Ollama) → in Qdrant speichern. Der AI-Agent greift dann automatisch auch auf diese Daten zu.

---

#### ELO (DMS / Dokumentenmanagement)

ELO bietet eine REST-API (`ELO REST API`) über die Dokumente, Akten und Metadaten abgerufen werden können.

**Anbindung über n8n:**

1. In n8n einen neuen Workflow anlegen (analog zu `op-api-speichern`)
2. Per **HTTP Request**-Knoten die ELO REST API abfragen:
   - Endpunkt: `http://<elo-server>/ix-<archivname>/plugin/de.elo.ix.plugin.proxy/rest/`
   - Authentifizierung: Basic Auth mit ELO-Benutzer
   - Relevante Endpunkte: `/search` für Volltextsuche, `/document/{id}` für einzelne Dokumente
3. Dokumenteninhalte und Metadaten (Titel, Ablageort, Datum) als Text zusammenführen
4. Über Ollama embedden und in eine eigene Qdrant-Collection speichern (z.B. `elo-documents`)
5. Im AI-Agent-Workflow ein weiteres **Vector Store Tool** für die `elo-documents`-Collection ergänzen

**Hinweis:** Für OCR-gescannte Dokumente in ELO muss der Volltext in ELO selbst bereits indiziert sein, damit er über die API abrufbar ist.

---

#### IBM DOORS (Anforderungsmanagement)

IBM DOORS Next (DOORS 9 mit DXL oder DOORS Next Generation mit REST) lässt sich ebenfalls als Datenquelle einbinden.

**DOORS Next Generation (DNG) – Anbindung über n8n:**

1. Neuen n8n-Workflow anlegen
2. Per **HTTP Request** die OSLC/REST-API von DNG abfragen:
   - Basis-URL: `https://<doors-server>/rm/`
   - Authentifizierung: Basic Auth oder OAuth
   - Relevante Ressourcen: Requirements, Module, Comments
3. Anforderungstext, ID, Modul und Status als strukturierten Text zusammenführen
4. Embedden und in Qdrant-Collection `doors-requirements` speichern
5. Im AI-Agent als weiteres Vector Store Tool ergänzen

**DOORS 9 (klassisch) – Anbindung über DXL-Export:**

Da DOORS 9 keine native REST-API bietet, empfiehlt sich ein regelmäßiger **DXL-Skript-Export** nach CSV oder JSON, der dann von n8n eingelesen wird:

1. DXL-Skript in DOORS 9 exportiert Anforderungen als CSV
2. n8n liest die CSV-Datei (z.B. aus einem Netzlaufwerk oder SFTP)
3. Weiterverarbeitung wie oben (Embeddings → Qdrant)

---

#### Allgemeines Schema für neue Datenquellen

Jede weitere Datenquelle folgt demselben Muster:

```
[Trigger: Schedule / Webhook]
    ↓
[HTTP Request: Daten aus System X abrufen]
    ↓
[Code-Knoten: Inhalt & Metadaten strukturieren]
    ↓
[Ollama Embedding: nomic-embed-text]
    ↓
[Qdrant: In eigene Collection speichern, z.B. "system-x"]
```

Im AI-Agent-Workflow dann ein weiteres **Vector Store Tool** für die neue Collection ergänzen – der Agent entscheidet selbst, welche Collections für eine Frage relevant sind.
