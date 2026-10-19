[English](README.md) · **Deutsch** · [Русский](README.ru.md)

# Telegram-Bot mit Gedächtnis: Dialog + RAG über Dokumente

Der Bot führt einen zusammenhängenden Dialog (Kurzzeitgedächtnis) und beantwortet Fragen zu hochgeladenen PDF-, DOCX- und TXT-Dateien
(Langzeitgedächtnis auf Basis von RAG und ChromaDB). Die Daten jedes Nutzers sind voneinander isoliert.

Die Oberfläche des Bots ist auf Russisch; die Nachrichten unten sind zur besseren Lesbarkeit übersetzt.

## Funktionen

- Dialog unter Berücksichtigung des Verlaufs: die letzten Nachrichten + eine kurze Zusammenfassung des älteren Gesprächsteils.
- Hochladen von PDF, DOCX, TXT → Textextraktion → überlappende Chunks → OpenAI-Embeddings → ChromaDB.
- Antworten zu Dokumenten ausschließlich auf Basis der gefundenen Fragmente, mit einem Block „Quellen in den Dokumenten".
- Dokumentverwaltung: Liste, Auswahl des aktiven Dokuments, Löschen mit Bestätigung.
- Dokumente und Index bleiben nach einem Neustart erhalten (das Kurzzeitgedächtnis wird geleert).

## Installation

Benötigt wird Python 3.11 oder neuer.

```bash
python -m venv .venv
source .venv/bin/activate      # macOS/Linux
.venv\Scripts\activate         # Windows PowerShell
pip install -r requirements.txt
cp .env.example .env           # in Windows PowerShell: Copy-Item .env.example .env
```

## .env einrichten

`.env` öffnen und die beiden Pflichtwerte eintragen:

| Variable | Bedeutung |
|---|---|
| `BOT_TOKEN` | Bot-Token von [@BotFather](https://t.me/BotFather) |
| `OPENAI_API_KEY` | Schlüssel von [platform.openai.com](https://platform.openai.com/api-keys) |

Die übrigen Variablen haben sinnvolle Standardwerte:

| Variable | Standard | Zweck |
|---|---|---|
| `CHAT_MODEL` | `gpt-5-mini` | Modell für Antworten und Zusammenfassungen |
| `EMBED_MODEL` | `text-embedding-3-small` | Embedding-Modell |
| `CHROMA_PATH` | `./data/chroma` | ChromaDB-Ordner |
| `UPLOADS_PATH` | `./data/uploads` | Ordner mit den Originaldateien (`{user_id}/`) |
| `SHORT_MEMORY_MESSAGES` | `10` | wie viele letzte Nachrichten an das Modell gehen |
| `SHORT_MEMORY_SUMMARY_AFTER` | `20` | nach wie vielen Nachrichten der Verlauf zusammengefasst wird (`0` – aus) |
| `CHUNK_SIZE` / `CHUNK_OVERLAP` | `500` / `75` | Chunk-Größe und Überlappung in Zeichen |
| `RAG_TOP_K` | `5` | wie viele Fragmente an das Modell übergeben werden |
| `MAX_DOCUMENT_SIZE_MB` | `20` | maximale Dateigröße |
| `RAG_MAX_DISTANCE` | `0.7` | Relevanzschwelle (Kosinus-Distanz; kleiner ist strenger) |
| `MAX_ANSWER_TOKENS` | `4000` | Token-Limit der Antwort (bei Reasoning-Modellen inklusive „Nachdenken") |
| `LOG_LEVEL` | `INFO` | Log-Level |

Pfade in `.env` sind relativ zum Projektordner. Die Datei `.env` wird nicht in git eingecheckt.

## Start

```bash
python bot.py
```

Im Log erscheint die Zeile „Бот запущен: @bot_name" (Bot gestartet). Beenden mit Strg+C.
Der Bot funktioniert nur in privaten Chats.

## Tests

```bash
pytest
```

## Bot-Befehle

| Befehl | Was er macht |
|---|---|
| `/start` | Begrüßung und kurze Hilfe |
| `/help` | alle Befehle und Erklärung des Gedächtnisses |
| `/documents` | Liste der Dokumente: Nummer, Datei, Datum, Anzahl Chunks, kurze ID |
| `/use_document 2` | nur in Dokument Nr. 2 suchen |
| `/all_documents` | wieder in allen Dokumenten suchen |
| `/delete_document 2` | Dokument Nr. 2 löschen (mit Bestätigung über „Löschen" / „Abbrechen") |
| `/clear_chat` | Gesprächsverlauf und dessen Zusammenfassung löschen (Dokumente bleiben) |

### Beispielsitzung

```
Sie:  /start
Sie:  [senden company.pdf]
Bot:  Datei empfangen. Text wird extrahiert und der Index erstellt…
Bot:  Fertig: company.pdf wurde ins Langzeitgedächtnis aufgenommen. Erstellte Chunks: 24. Sie können jetzt Fragen stellen.
Sie:  Machen Sie Webentwicklung?
Bot:  Ja, ...
      Quellen in den Dokumenten: company.pdf, Fragmente 2 und 4
Sie:  Und wie lange dauert das?   ← „das" ergibt sich aus dem Gesprächsverlauf
Bot:  In den hochgeladenen Dokumenten habe ich dafür keine Bestätigung gefunden.
Sie:  /clear_chat
Bot:  Der Verlauf des aktuellen Gesprächs wurde gelöscht
```

## Aufbau

```
bot.py                     Telegram-Handler (aiogram), Start, Logging
config.py                  Lesen und Prüfen der .env
services/
  document_loader.py       sicherer Dateiname, Textextraktion (TXT/PDF/DOCX)
  chunker.py               chunk_text(): Chunks nach Absätzen/Sätzen mit Überlappung
  embeddings.py            OpenAI-Embeddings in Batches
  vector_store.py          ChromaDB: Collection long_memory, Kosinus-Ähnlichkeit
  registry.py              SQLite data/app.db: Tabellen documents und user_settings
  documents.py             Pipeline „Datei → Index" und Löschen eines Dokuments
  short_memory.py          Verlauf und Zusammenfassung pro user_id (deque im Arbeitsspeicher)
  rag.py                   retrieve_context(), Aufbau des Prompts, Modellaufruf, Zusammenfassung
tests/                     pytest
data/uploads/{user_id}/    Originaldateien
data/chroma/               Vektordatenbank
data/app.db                Dokumentenregister
```

### Kurzzeitgedächtnis

- Der Verlauf wird pro `user_id` in einer `deque` gespeichert; Nachricht des Nutzers und Antwort des Assistenten zählen je als eine Nachricht.
- An das Modell gehen die letzten `SHORT_MEMORY_MESSAGES` Nachrichten.
- Damit die Zusammenfassung greifen kann (die Schwelle 20 ist größer als das Fenster 10), fasst die `deque` `max(SHORT_MEMORY_MESSAGES, SHORT_MEMORY_SUMMARY_AFTER + 2)` Nachrichten; bei Überlauf wird die älteste verdrängt (FIFO).
- Sobald es mehr als `SHORT_MEMORY_SUMMARY_AFTER` Nachrichten sind, erstellt das Modell eine Zusammenfassung (Fakten, Ziele, Entscheidungen; ohne Geheimnisse), im Verlauf bleiben die letzten 6 Nachrichten. Die Zusammenfassung wird dem System-Prompt hinzugefügt. Schlägt die Zusammenfassung fehl, bleibt der Verlauf unverändert und der Versuch wird später wiederholt.

### Langzeitgedächtnis und Datenisolation

- Alle Chunks liegen in einer Collection `long_memory`, aber **jede Suche, Zählung und Löschung** enthält einen Filter auf `user_id` (siehe `_where()` in `services/vector_store.py`); bei ausgewähltem Dokument kommt `doc_id` hinzu.
- Auch das Dokumentenregister und das Löschen prüfen `user_id`, sodass das Dokument eines anderen Nutzers weder gefunden noch gelöscht werden kann – selbst wenn man dessen `doc_id` einsetzt.
- Die Suche verwirft Ergebnisse jenseits der Schwelle `RAG_MAX_DISTANCE` und entfernt nahezu identische Chunks. Findet sich nichts Relevantes, erhält das Modell einen entsprechenden Hinweis und antwortet „In den hochgeladenen Dokumenten habe ich dafür keine Bestätigung gefunden."
- Eine kurze Rückfrage („Und wie lange dauert das?") wird zusammen mit der vorherigen Frage des Nutzers gesucht.
- `/use_document` beschränkt die Suche strikt auf das gewählte Dokument: Wird darin nichts gefunden, weicht der Bot nicht auf andere Dokumente aus. Die Auswahl wird in SQLite gespeichert und übersteht einen Neustart; zurückgesetzt wird sie mit `/all_documents` oder durch Löschen des Dokuments.

## Sicherheit und Datenschutz

- Der Dateiname wird von Pfaden und Sonderzeichen bereinigt; die Datei wird als `data/uploads/{user_id}/{doc_id}_{name}` gespeichert.
- Dateiendung und -größe werden geprüft; Nutzerdaten werden niemals an eine Shell übergeben.
- Der Nutzer sieht nur verständliche Meldungen; Tracebacks und Fehlerdetails landen im Log.
- Die Logs enthalten keine Dokumentinhalte und keine Nachrichtentexte; Bot-Token und OpenAI-Schlüssel werden maskiert (`***`).
- Die Texte Ihrer Dokumente und Fragen werden an OpenAI gesendet – bitte beim Hochladen vertraulicher Daten berücksichtigen.

## Häufige Probleme

- **„Не задана переменная окружения BOT_TOKEN/OPENAI_API_KEY"** (Umgebungsvariable nicht gesetzt) – `.env` ausfüllen; die Platzhalter aus `.env.example` funktionieren nicht.
- **PDF ohne Text** – es handelt sich um einen Scan; der Bot führt keine OCR durch, bitte ein PDF mit Textebene hochladen.
- **`EMBED_MODEL` nach der Indexierung geändert** – Modelle haben unterschiedliche Vektordimensionen, alte Dokumente werden dann nicht mehr gefunden. Mit `/delete_document` löschen (oder den Ordner `data/chroma` zusammen mit `data/app.db` leeren) und erneut hochladen.
- **Das Modell antwortet zu langsam** – `gpt-5-mini` denkt vor der Antwort nach; für mehr Tempo in `CHAT_MODEL` ein weniger „nachdenkendes" Modell eintragen.
