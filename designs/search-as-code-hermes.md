# Search as Code (SaC) für Hermes — Analyse & Designentwurf

Status: Entwurf · Sprache: DE · Zielsystem: Hermes (Agent-Runtime mit
`web_search`, `web_extract`, `execute_code`, `memory`, Skill- und Cron-System)

Dieses Dokument prüft den eingereichten Entwurf „Search as Code für Hermes"
auf **Umsetzbarkeit** und **Schwächen** und stellt anschließend einen
**vollständig überarbeiteten Designentwurf** mit korrigierter
Referenzimplementierung bereit.

---

## 1. TL;DR

Die Grundidee ist richtig und übertragbar: **Strategie (LLM) von Ausführung
(deterministischer Code) trennen** und das Context-Window schonen, indem nur
das gefilterte Endergebnis zum Modell zurückfließt.

Der konkrete Entwurf ist aber **noch nicht lauffähig** und steht und fällt mit
**einer einzigen, ungeprüften Annahme**:

> **Kann der `execute_code`-Sandbox `web_search`/`web_extract` als
> aufrufbare Python-Funktionen erreichen — und hat er dafür Netzzugang?**

Solange das nicht verifiziert ist, ist alles andere Spekulation. Hinzu kommen
mehrere **echte Bugs**, die schon vor jeder Architekturfrage dafür sorgen, dass
die Funktion in der Praxis **fast immer leere oder falsch sortierte Ergebnisse**
liefert (zu strenger Filter, kaputtes Rerank-Scoring, fehlerhaftes
Index-Mapping bei der Extraktion). Die behaupteten „~85 % Tokenersparnis" gelten
**nur**, wenn die Tools wirklich in der Sandbox laufen — sonst ist die Ersparnis
gleich null oder negativ.

Der überarbeitete Entwurf in Abschnitt 5 behebt das: klares
Machbarkeits-Gate, zwei tragfähige Architekturvarianten, korrigierte Pipeline
(Reihenfolge + Filter + Rerank + URL-basiertes Mapping), zweischichtiges
Caching, ein zustandssicheres Skill-/Modul-Modell und ehrliche
Tokenrechnung.

---

## 2. Ausgangsentwurf — Kurzreferenz

Der Entwurf bildet Perplexitys SaC-Primitive auf Hermes-Bausteine ab:

| SaC-Primitiv            | Hermes-Baustein         |
| ----------------------- | ----------------------- |
| `retrieve`              | `web_search`            |
| `parse_field`           | `web_extract`           |
| Sandbox + `filter/dedupe/rerank/fanout` | `execute_code` (Python) |
| Zwischencache           | `memory` / Dateien      |
| wiederverwendbare Vorlage | Skill-System          |

Vorgeschlagen wird eine Funktion `search_as_code(...)`, die Fanout → Retrieve →
Extract → Filter → Dedupe → Rerank → Cache durchläuft, als Skill abgelegt und
per Cron-Job für wiederholtes Monitoring aufgerufen wird.

Die **Idee** ist gut. Die **Ausführung** hat die folgenden Probleme.

---

## 3. Machbarkeitsanalyse

### 3.1 GATE — Tool-Zugriff aus der Sandbox (entscheidend)

Der Entwurf schreibt `from hermes_tools import web_search, web_extract` **in den
Code, der über `execute_code` läuft**. Das setzt voraus, dass:

1. der Sandbox-Interpreter ein Python-Modul `hermes_tools` mit diesen Funktionen
   bereitstellt, **und**
2. diese Funktionen aus der Sandbox heraus **Netzwerkzugriff** haben (oder über
   einen Broker an den Host delegieren dürfen).

In den meisten Agent-Harnesses sind `web_search`/`web_extract` jedoch
**Agent-Level-Tools**, die vom Orchestrator/LLM aufgerufen werden — **nicht**
als Bibliothek im Code-Sandbox verfügbar. Viele Code-Sandboxes sind aus
Sicherheitsgründen zudem **netzwerkisoliert**. Trifft das auf Hermes zu, ist der
gesamte Entwurf so **nicht lauffähig**.

Der Entwurf widerspricht sich hier außerdem selbst: In Abschnitt „Sicherheit"
heißt es „keine zusätzliche Isolation" — gerade *das* wäre die Voraussetzung für
Netzzugang, untergräbt aber das Sicherheitsversprechen von SaC (dessen
Sicherheit in Perplexity *gerade* aus der starken Sandbox kommt).

**→ Diese Frage muss zuerst beantwortet werden** (siehe Verifikations-Checkliste,
Abschnitt 6). Abschnitt 5.1 liefert für *beide* Antworten eine tragfähige
Architektur.

### 3.2 Zustand zwischen `execute_code`-Läufen

Das Nutzungsbeispiel ruft `execute_code` zweimal auf: erst „definiert die
Funktion im Kontext", dann „nehmen wir an, die Funktion ist bereits geladen".
Das setzt einen **persistenten REPL** voraus, in dem Definitionen über Aufrufe
hinweg überleben.

Viele `execute_code`-Implementierungen sind **zustandslos** (frischer Prozess
pro Aufruf). Dann schlägt der zweite Aufruf mit `NameError: search_as_code`
fehl — und ebenso `json`/`time`, die im zweiten Snippet benutzt, aber nur im
ersten importiert werden. Der überarbeitete Entwurf macht **keine Annahme über
REPL-Persistenz** (Abschnitt 5.4).

### 3.3 Tokenersparnis — realistisch?

Perplexitys ~85 % stammen aus einem **in-Sandbox-SDK**: Primitive laufen im
Prozess, nur das kompakte Endergebnis geht ins Modell.

- **Wenn** Hermes-Tools in-Sandbox aufrufbar sind (3.1 = ja): Ersparnis
  plausibel, denn Such-Roundtrips umgehen das LLM-Context-Window.
- **Wenn nicht** (Tools sind Agent-Level): Jeder `web_search`/`web_extract`-Call
  ist ein eigener Tool-Roundtrip durch den Orchestrator — die Rohtreffer landen
  trotzdem im Context. Dann ist die Ersparnis **null**, ggf. **negativ** (zzgl.
  Code-Generierung). Die Aussage „nahezu null Token-Verbrauch" ist in diesem
  Fall falsch.

Ehrliche Formel (Abschnitt 5.8): Ersparnis entsteht nur durch (a) In-Sandbox-
Ausführung **und** (b) Rückgabe ausschließlich des verdichteten Ergebnisses
**und** (c) Cache-Treffer bei Wiederholung.

---

## 4. Schwächen im Detail

Sortiert nach Schweregrad. „Blocker" = verhindert korrekte Funktion schon ohne
die Architekturfrage aus 3.1.

| # | Schwere | Fundstelle | Problem | Fix (Abschnitt 5) |
|---|---------|-----------|---------|-------------------|
| 1 | **Blocker** | `is_relevant` | Verlangt das **gesamte** `base_query` als Teilstring in Titel/Snippet. Bei Mehrwort-Queries („Perplexity Search as Code Tokenersparnis") trifft das praktisch nie → **fast immer leeres Ergebnis**. | Token-Overlap mit Schwellwert statt Substring (5.2). |
| 2 | **Blocker** | Rerank-Score | Belohnt **Länge** von Titel/Snippet (`+= len(...)`) → bevorzugt geschwätzige/Spam-Treffer. `content.count(base_query)` zählt die **ganze Phrase**, die fast nie vorkommt → Beitrag ≈ 0. Effektiv eine Längen-Sortierung. | BM25-naher, längennormierter Term-Score (5.2). |
| 3 | **Blocker** | `web_extract`-Mapping | Mappt `extracted.results[i] → raw_results[i]` **positionell**. Wenn Extraktion eine URL überspringt/umsortiert, landet Content am **falschen** Treffer. | Mapping per **URL-Schlüssel**, nicht Index (5.2/5.5). |
| 4 | Hoch | Pipeline-Reihenfolge | Extraktion läuft **vor** Filter/Dedupe → teure `web_extract`-Calls für Treffer, die danach verworfen werden (widerspricht dem eigenen „Kostenkontrolle"-Ziel). | Reihenfolge: Dedupe → Filter → Rerank → **nur Top-K extrahieren** (5.1). |
| 5 | Hoch | Fanout | „Fanout" = Anhängen von `site:de/org/edu`. `site:org` ist **kein gültiger** TLD-Filter (erwartet Domain), und die Heuristik **verwirft** `.com/.net/...` komplett. Das ist Quellen-Restriktion, **kein** semantischer Fanout. | Echte Query-Variation/Decomposition; TLD nur als optionaler Zusatz (5.2). |
| 6 | Hoch | 3.1 (s. o.) | In-Sandbox-Toolzugriff unbelegt. | Gate + zwei Architekturen (5.1). |
| 7 | Mittel | 3.2 (s. o.) | Verlässt sich auf REPL-Persistenz. | Zustandssicheres Lade-Modell (5.4). |
| 8 | Mittel | URL-Dedupe | `uid = url or title` ohne **Normalisierung** → `http`/`https`, Trailing-Slash, `utm_*`-Parameter erzeugen Dubletten. | URL-Normalisierung vor Dedupe (5.5). |
| 9 | Mittel | Cache | Schreibt nach `~/.hermes/search_cache/{key}.json` ohne **Verzeichnis-Anlage** / Fehlerbehandlung; `~`-Expansion durch `memory` unbestätigt. Cache-Key ignoriert `rerank`/`dedupe`. | Robustes, zweischichtiges Caching (5.3). |
| 10 | Mittel | Output | `print(...)`-Warnungen landen im `execute_code`-**Output**, den das Modell liest → bläht Context auf und untergräbt die Tokenersparnis. | Strukturiertes Log-Feld statt stdout (5.5/5.8). |
| 11 | Niedrig | Cron/Skill-Schema | `cron/jobs/*.yml`-Format, `deliver: telegram`, Modell `nvidia/nemotron-3-super-120b-a12b:free` sind **angenommen** und teils fragwürdig (Modell-ID wirkt verstümmelt). | Als zu verifizierende Annahmen markiert (6). |
| 12 | Niedrig | „parallel" | Kommentar sagt „parallel", Code ist **sequentiell** → Latenz ∝ Quellen × Extraktionen. | Optionaler Fanout via `delegate_task`, sonst seriell (5.6). |
| 13 | Niedrig | Determinismus | „Deterministische Filterung" stimmt fürs Post-Processing, aber `web_search` selbst ist nichtdeterministisch; der Nutzen wird durch die schwachen Filter (#1/#2) zunichtegemacht. | Bessere Filter + Cache als Stabilisator (5.2/5.3). |

---

## 5. Überarbeiteter Designentwurf

### 5.1 Architektur & Datenfluss

Erst das Gate aus 3.1 entscheiden, dann eine Variante wählen:

**Variante A — Tools sind in-Sandbox aufrufbar (bevorzugt, voller SaC-Nutzen).**
Der gesamte Orchestrierungscode läuft in `execute_code`; nur das verdichtete
Ergebnis (Top-K, schlanke Felder) geht zum Modell zurück.

**Variante B — Tools sind Agent-Level (kein In-Sandbox-Netz).**
Hybrid: Der Code übernimmt **deterministische, datenlokale** Schritte (Dedupe,
Filter, Rerank, Caching, URL-Normalisierung). Die **Tool-Calls**
(`web_search`/`web_extract`) macht der Agent außen herum; Roh- und
Zwischenergebnisse werden über `memory`/Dateien gereicht, nicht über den
Prompt. Tokenersparnis kommt dann v. a. aus **Cache** + **Verdichtung vor der
Modellrückgabe**, nicht aus dem Umgehen der Tool-Roundtrips.

Korrigierter Datenfluss (beide Varianten):

```
fanout(base_query) → [sub_queries]
   → retrieve (web_search je sub_query)        # Rohtreffer sammeln
   → merge + URL-Normalisierung
   → dedupe (nach normalisierter URL)          # früh, spart Arbeit
   → filter (Token-Overlap ≥ Schwelle)         # vor teurer Extraktion
   → rerank (längennormiert, Term-Coverage)
   → select Top-K
   → extract NUR Top-K (web_extract, URL-Map)  # teuer, daher minimal
   → optional: re-rank mit Volltext-Signal
   → cache (zweischichtig) + return schlanke Treffer
```

Kernänderung gegenüber dem Entwurf: **Extraktion ganz nach hinten**, nur für die
bereits gefilterten Top-K — das ist der größte Kostenhebel.

### 5.2 Pipeline-Stufen (korrigiert)

- **Fanout:** Standardmäßig leichte, deterministische Query-Varianten
  (z. B. Synonyme/Reformulierungen aus einer übergebenen Liste). TLD/`site:`
  nur als *optionaler* Zusatz, nie als Ersatz für `.com` & Co. Optional einmalig
  modellgeneriert (siehe 5.8: einmaliger Strategie-Call).
- **Filter:** Query in Terme zerlegen; ein Treffer passt, wenn die
  **Term-Abdeckung** in Titel+Snippet(+Content) ≥ Schwellwert (z. B. 0,5) ist.
  Kein Ganz-Phrasen-Substring mehr.
- **Rerank:** Transparenter Heuristik-Score, **längennormiert**:
  Term-Coverage (Anteil getroffener Query-Terme) als Hauptsignal,
  Term-Frequenz normiert auf Dokumentlänge als Nebensignal, optionaler
  Domain-Prior. Klarer Hook, um bei Bedarf auf **Embedding-Rerank**
  umzustellen, falls Hermes Embeddings bereitstellt.

### 5.3 Caching (zweischichtig)

1. **Query-Ergebnis-Cache** — Key aus normalisierter Query-Menge **und allen
   ergebnisrelevanten Parametern** (`sources`, `n`, `dedupe`, `rerank`,
   Filter-Schwelle). TTL pro Thema (zeitkritisch: 30 min; statisch: Stunden).
2. **URL-Content-Cache** — Key = normalisierte URL. So werden teure
   `web_extract`-Ergebnisse **über verschiedene Queries hinweg** wiederverwendet
   (großer Hebel beim Monitoring überlappender Themen).

Schreiboperationen mit Verzeichnis-Anlage und `try/except` absichern; keine
Annahme über `~`-Expansion (absolute Pfade verwenden).

### 5.4 Zustands-/Skill-Modell (zustandssicher)

**Keine** Abhängigkeit von REPL-Persistenz. Stattdessen:

- Code einmalig als Datei/Modul ablegen (Skill oder
  `<hermes-home>/lib/search_as_code.py`).
- Jeder `execute_code`-Aufruf macht nur einen **kleinen Bootstrap**:
  `sys.path` erweitern, `from search_as_code import search_as_code`, dann
  aufrufen. Ein Roundtrip, keine erneute Übertragung des Volltexts.
- Falls der Sandbox **weder** Dateien persistiert **noch** REPL-Zustand hält:
  Code einmal inline laden (Kosten akzeptieren) und das **Ergebnis** cachen — die
  Wiederholung profitiert dann vom Cache, nicht von der Code-Persistenz.

### 5.5 Referenzimplementierung (korrigiert)

> Lauffähigkeit hängt vom Gate (3.1) ab. Tool-Signaturen
> (`web_search`/`web_extract`/`memory`) ggf. an die echte Hermes-API anpassen.
> Der Code macht **keine** Annahme über REPL-Persistenz und schreibt Warnungen
> in ein **Log-Feld** statt nach stdout.

```python
"""search_as_code.py — Search-as-Code-Orchestrierung für Hermes.

Pipeline: fanout -> retrieve -> normalize -> dedupe -> filter -> rerank
          -> select Top-K -> extract (nur Top-K) -> (re-rank) -> cache.
"""
from __future__ import annotations
import hashlib, json, os, re, time
from urllib.parse import urlsplit, urlunsplit, parse_qsl, urlencode

# In Hermes anzupassen / zu verifizieren (siehe Gate 3.1):
from hermes_tools import web_search, web_extract, memory

CACHE_DIR = "/root/.hermes/search_cache"          # absoluter Pfad, kein "~"
CONTENT_DIR = "/root/.hermes/content_cache"
_TRACKING = re.compile(r"^(utm_|fbclid$|gclid$|mc_eid$|ref$)")


# ---------- Hilfsfunktionen ------------------------------------------------
def _terms(text: str) -> list[str]:
    return [t for t in re.split(r"\W+", text.lower()) if len(t) > 1]


def normalize_url(url: str) -> str:
    if not url:
        return ""
    s = urlsplit(url.strip())
    scheme = "https" if s.scheme in ("http", "https", "") else s.scheme
    netloc = s.netloc.lower()
    path = s.path.rstrip("/") or "/"
    query = urlencode([(k, v) for k, v in parse_qsl(s.query)
                       if not _TRACKING.match(k)])
    return urlunsplit((scheme, netloc, path, query, ""))  # Fragment verwerfen


def _coverage(query_terms: set[str], *texts: str) -> float:
    if not query_terms:
        return 1.0
    blob = " ".join(t.lower() for t in texts if t)
    hit = sum(1 for qt in query_terms if qt in blob)
    return hit / len(query_terms)


def _read_json(path: str) -> dict | None:
    try:
        c = memory.read(path=path)
        return json.loads(c["content"]) if c and c.get("content") else None
    except Exception:
        return None


def _write_json(path: str, obj: dict) -> None:
    try:
        os.makedirs(os.path.dirname(path), exist_ok=True)
        memory.write(path=path, content=json.dumps(obj, ensure_ascii=False))
    except Exception:
        pass  # Cache ist best-effort, nie Fehlerquelle


# ---------- Pipeline-Stufen ------------------------------------------------
def fanout(base_query: str, variants: list[str] | None,
           tld_sites: list[str] | None) -> list[str]:
    queries = [base_query] + [f"{base_query} {v}" for v in (variants or [])]
    if tld_sites:                       # optionaler Zusatz, kein Ersatz
        queries += [f"{base_query} site:{d}" for d in tld_sites]
    return list(dict.fromkeys(queries))  # Reihenfolge erhalten, dedupe


def rerank_score(query_terms: set[str], r: dict) -> float:
    title, snippet = r.get("title", ""), r.get("snippet", "")
    content = r.get("content", "")
    cov = _coverage(query_terms, title, snippet, content)        # Hauptsignal
    blob_terms = _terms(f"{title} {snippet} {content}")
    tf = (sum(t in query_terms for t in blob_terms) /
          (len(blob_terms) + 1)) if blob_terms else 0.0          # längennormiert
    title_cov = _coverage(query_terms, title)                    # Titel-Bonus
    return 3.0 * cov + 1.0 * tf + 0.5 * title_cov


# ---------- Hauptfunktion --------------------------------------------------
def search_as_code(
    base_query: str,
    query_variants: list[str] | None = None,
    tld_sites: list[str] | None = None,
    max_results_per_query: int = 5,
    top_k: int = 10,
    extract_top_k: int = 5,
    min_coverage: float = 0.5,
    cache_ttl_sec: int = 3600,
) -> dict:
    log: list[str] = []
    qterms = set(_terms(base_query))

    # 0) Query-Ergebnis-Cache -------------------------------------------------
    ckey = hashlib.sha256(json.dumps({
        "q": base_query, "v": query_variants, "t": tld_sites,
        "n": max_results_per_query, "k": top_k, "e": extract_top_k,
        "mc": min_coverage,
    }, sort_keys=True).encode()).hexdigest()
    cpath = f"{CACHE_DIR}/{ckey}.json"
    cached = _read_json(cpath)
    if cached and time.time() - cached.get("_ts", 0) < cache_ttl_sec:
        return {"results": cached["results"], "cached": True, "log": []}

    # 1) Fanout + 2) Retrieve -------------------------------------------------
    raw: list[dict] = []
    for sq in fanout(base_query, query_variants, tld_sites):
        try:
            resp = web_search(query=sq, limit=max_results_per_query)
            for r in resp.get("data", {}).get("web", []):
                raw.append({"title": r.get("title", ""),
                            "url": r.get("url", ""),
                            "snippet": r.get("description", ""),
                            "source": sq})
        except Exception as e:
            log.append(f"web_search fehlgeschlagen für {sq!r}: {e}")

    # 3) Normalisieren + 4) Dedupe (nach normalisierter URL) ------------------
    seen, items = set(), []
    for r in raw:
        nu = normalize_url(r["url"])
        key = nu or r["title"]
        if key and key not in seen:
            seen.add(key)
            r["norm_url"] = nu
            items.append(r)

    # 5) Filter (Token-Coverage, KEIN Ganz-Phrasen-Substring) -----------------
    items = [r for r in items
             if _coverage(qterms, r["title"], r["snippet"]) >= min_coverage]

    # 6) Rerank (vor Extraktion) + 7) Top-K -----------------------------------
    for r in items:
        r["score"] = rerank_score(qterms, r)
    items.sort(key=lambda x: x["score"], reverse=True)
    items = items[:top_k]

    # 8) Extraktion NUR der Top-K, Mapping per URL (nicht per Index) ----------
    to_extract = [r for r in items[:extract_top_k] if r["norm_url"]]
    if to_extract:
        by_url = {}
        missing = []
        for r in to_extract:                       # URL-Content-Cache zuerst
            hit = _read_json(f"{CONTENT_DIR}/"
                             f"{hashlib.sha256(r['norm_url'].encode()).hexdigest()}.json")
            if hit and time.time() - hit.get("_ts", 0) < cache_ttl_sec:
                by_url[r["norm_url"]] = hit["content"]
            else:
                missing.append(r["norm_url"])
        if missing:
            try:
                ext = web_extract(urls=missing)
                for e in ext.get("results", []):       # per URL zuordnen
                    nu = normalize_url(e.get("url", ""))
                    by_url[nu] = e.get("content", "")
                    _write_json(f"{CONTENT_DIR}/"
                                f"{hashlib.sha256(nu.encode()).hexdigest()}.json",
                                {"_ts": time.time(), "content": by_url[nu]})
            except Exception as e:
                log.append(f"web_extract fehlgeschlagen: {e}")
        for r in items:
            if r["norm_url"] in by_url:
                r["content"] = by_url[r["norm_url"]]
        # 9) Optionales Re-Rank mit Volltext-Signal
        for r in items:
            r["score"] = rerank_score(qterms, r)
        items.sort(key=lambda x: x["score"], reverse=True)

    # 10) Schlanke Rückgabe + Cache ------------------------------------------
    slim = [{"title": r["title"], "url": r["norm_url"] or r["url"],
             "snippet": r["snippet"], "score": round(r["score"], 3)}
            for r in items]
    _write_json(cpath, {"_ts": time.time(), "results": slim})
    return {"results": slim, "cached": False, "log": log}
```

Wesentliche Verbesserungen gegenüber dem Original: korrekte Reihenfolge
(Extraktion zuletzt, nur Top-K), Token-Coverage-Filter statt
Ganz-Phrasen-Substring, längennormiertes Rerank, **URL-basiertes**
Extract-Mapping, URL-Normalisierung, zweischichtiger Cache, Log-Feld statt
stdout, absolute Pfade, best-effort Cache ohne Fehlerquelle.

### 5.6 Scheduling / Monitoring

- Cron ruft ein dünnes Skill auf, das `search_as_code(...)` lädt (5.4) und das
  Ergebnis ausführt; nur eine **kompakte Zusammenfassung** geht an einen
  kurzen LLM-Schritt → wenige Tokens.
- TTL pro Thema steuern (zeitkritisch kurz, statisch lang).
- **Optionale** Parallelisierung großer Fanouts via `delegate_task` (ein
  Sub-Agent je `web_search`) — nur wenn die Quellenzahl die serielle Latenz zum
  Problem macht; der Overhead ist sonst nicht gerechtfertigt.
- Konkrete YAML/Schema-Felder (`schedule`, `deliver`, `model`) sind **vor**
  Gebrauch gegen die echte Hermes-Cron-Doku zu prüfen (siehe 6). Die im
  Original genannte Modell-ID `nvidia/nemotron-3-super-120b-a12b:free` ist zu
  verifizieren — sie wirkt verstümmelt; nicht ungeprüft übernehmen.

### 5.7 Sicherheit

- SaC ist nur so sicher wie der Sandbox. Falls Hermes „keine zusätzliche
  Isolation" bietet und der Sandbox Host-Netz/-FS/Secrets sieht, ist das
  Ausführen generierten Codes ein reales Risiko.
- Empfehlung: Skill-Code **versionieren und reviewen**; generierten Ad-hoc-Code
  nicht ungeprüft ausführen; Cache-Pfade auf ein dediziertes Verzeichnis
  beschränken; keine Secrets in Cache-Dateien schreiben.
- `web_extract`-Ziele sind nicht vertrauenswürdig — extrahierten Content nur als
  Daten behandeln, nie als Anweisungen (Prompt-Injection beim
  Zusammenfassungsschritt).

### 5.8 Observability & ehrliche Kostenrechnung

- **Tokenersparnis-Modell:** Ersparnis = (vermiedene Roh-Treffer im Context) +
  (Cache-Treffer). Sie ist real in Variante A, gering/keine in Variante B ohne
  Cache. Nicht pauschal „85 %" behaupten — das ist Perplexitys Zahl, nicht
  Hermes'. Vor/nach mit echten Token-Zählern messen.
- **Einmaliger Strategie-Call:** Query-Varianten können einmalig modellgeneriert
  und im Skill/Cache abgelegt werden; danach ist der Lauf rein Tool-/Code-basiert.
- **Logs** ins Rückgabe-`log`-Feld, nicht nach stdout — so bläht Debugging den
  Context nicht auf. Optional Metriken (Treffer pro Stufe, Cache-Hit-Rate)
  persistieren.

---

## 6. Verifikations-Checkliste (vor dem Bau)

Reihenfolge = Priorität. Punkt 1 ist das Gate.

1. **[GATE]** Kann `execute_code` `web_search`/`web_extract` als Funktionen
   aufrufen, und hat der Sandbox Netzzugang? → entscheidet Variante A vs. B.
2. Ist der `execute_code`-Zustand **persistent** über Aufrufe oder zustandslos?
   → entscheidet das Lade-Modell (5.4).
3. Echte Signaturen/Rückgabeformate von `web_search` (`data.web[*]`?) und
   `web_extract` (Feld `results[*].url/content`? Reihenfolge garantiert?).
4. `memory`-API: Schlüssel- vs. Pfadsemantik, `~`-Expansion,
   Verzeichnis-Anlage, Größenlimits, TTL/Quota.
5. Cron-System: existiert es, exaktes YAML-Schema, `deliver`-Kanäle,
   gültige Modell-IDs (die Original-ID prüfen/ersetzen).
6. Skill-System: `skill_manage`/`skill_view`-Verhalten, ob ein Skill Python-Code
   *bereitstellen* (importierbar machen) statt nur Text liefern kann.
7. Sandbox-Sicherheitsmodell: Netz, FS-Scope, Secret-Exposition.
8. Verfügbarkeit von Embeddings (für optionalen Rerank-Upgrade).

---

## 7. Umsetzungs-Roadmap

1. **Verifikation** (Abschnitt 6, v. a. Gate + Zustand).
2. **MVP**: `search_as_code` mit korrigiertem Filter/Rerank/Mapping, einschichtigem
   Query-Cache, an *einer* realen Query messen (Treffergüte + Tokens vorher/nachher).
3. **Cache-Layer 2** (URL-Content-Cache) + Skill-Verpackung (5.4).
4. **Scheduling** erst nach bestätigtem Cron-Schema; mit kurzem
   Zusammenfassungs-Schritt + Zustellkanal.
5. **Optimierung**: optionaler Embedding-Rerank, `delegate_task`-Fanout nur bei
   nachgewiesenem Latenzbedarf.

---

## 8. Fazit

Das Paradigma — **Strategie vom Ausführen trennen, nur Verdichtetes ins Modell**
— ist für Hermes sinnvoll und lohnend. Der eingereichte Entwurf ist als
*Konzeptskizze* gut, als *Implementierung* aber durch drei Blocker-Bugs
(Filter, Rerank, Extract-Mapping) und eine ungeprüfte Kernannahme (In-Sandbox-
Toolzugriff) noch nicht tragfähig, und die „85 %"-Ersparnis ist nicht
übertragbar behauptet.

Der überarbeitete Entwurf behebt die Bugs, stellt die Pipeline auf die
kosteneffiziente Reihenfolge um (Extraktion zuletzt, nur Top-K), macht das
Zustands-/Cache-Modell robust und ersetzt Marketingzahlen durch eine messbare
Kostenrechnung. **Nächster Schritt: das Gate aus Abschnitt 6.1 verifizieren** —
danach geradeaus über die Roadmap.
