# Search as Code (SaC) für Hermes — Analyse & Designentwurf

Status: Entwurf · Sprache: DE · Zielsystem: Hermes Agent (NousResearch)

Dieses Dokument prüft den eingereichten Entwurf „Search as Code für Hermes"
auf **Umsetzbarkeit** und **Schwächen** und stellt anschließend einen
**vollständig überarbeiteten Designentwurf** mit korrigierter
Referenzimplementierung bereit.

Die Aussagen zur Hermes-Sandbox in Abschnitt 3 sind gegen den **konkreten
Code/Doku** des Harnesses verifiziert
(`github.com/NousResearch/hermes-agent`), insbesondere
`website/docs/user-guide/features/code-execution.md` und `tools/web_tools.py`.
Verifizierte Fakten sind als **[verifiziert]** markiert, offene Punkte als
**[zu prüfen]**.

---

## 1. TL;DR

Die Grundidee ist richtig und für Hermes **sogar nativ vorgesehen**: Hermes'
„Code Execution / Programmatic Tool Calling" *ist* genau das SaC-Muster —
**Strategie (LLM) von Ausführung (deterministischer Code) trennen**, Tools im
Code aufrufen und nur das verdichtete Ergebnis ins Context-Window zurückgeben.

Damit ist die **Kernannahme des Entwurfs bestätigt** (nicht, wie ich zunächst
vermutete, das Hauptrisiko): `from hermes_tools import web_search, web_extract`
funktioniert, Tool-Ergebnisse bleiben im Skript, und „**only the final
`print()` output comes back, dramatically reducing token usage**"
**[verifiziert]**. Die behauptete große Tokenersparnis ist hier also
plausibel — sie ist der erklärte Zweck des Features.

Trotzdem ist der konkrete Code **noch nicht lauffähig**. Es bleiben:

1. **Drei Blocker-Bugs** in der Logik (zu strenger Filter, kaputtes
   Rerank-Scoring, positionelles statt URL-basiertes Extract-Mapping) → in der
   Praxis fast immer leere oder falsch sortierte Ergebnisse.
2. **Falsche API-Annahmen** gegen den echten Hermes-Code:
   `from hermes_tools import memory` existiert **nicht** (Cache muss über
   `read_file`/`write_file` laufen); `web_extract` erlaubt **max. 5 URLs**
   (Entwurf nutzt 10); Tools liefern **JSON-Strings**; der Code-Child läuft mit
   **minimaler Umgebung ohne Secrets**.
3. **Zustandslosigkeit**: Der Child-Prozess ist pro Aufruf frisch, das
   Staging-Verzeichnis wird gelöscht. Das „Funktion in Aufruf 1 definieren, in
   Aufruf 2 nutzen"-Muster des Entwurfs schlägt fehl.

Abschnitt 5 behebt all das mit einer korrigierten Pipeline, korrekter Hermes-
API-Nutzung, zustandssicherem Lade-Modell und zweischichtigem Datei-Caching.

---

## 2. Ausgangsentwurf — Kurzreferenz

Der Entwurf bildet Perplexitys SaC-Primitive auf Hermes-Bausteine ab:

| SaC-Primitiv            | Hermes-Baustein         |
| ----------------------- | ----------------------- |
| `retrieve`              | `web_search`            |
| `parse_field`           | `web_extract`           |
| Sandbox + `filter/dedupe/rerank/fanout` | `execute_code` (Python) |
| Zwischencache           | ~~`memory`~~ → `read_file`/`write_file` |
| wiederverwendbare Vorlage | Skill-System          |

Vorgeschlagen wird eine Funktion `search_as_code(...)`, die Fanout → Retrieve →
Extract → Filter → Dedupe → Rerank → Cache durchläuft, als Skill abgelegt und
per Cron-Job für wiederholtes Monitoring aufgerufen wird. Das Mapping ist im
Prinzip korrekt — bis auf den Cache (`memory` gibt es im Code-Stub nicht).

---

## 3. Machbarkeitsanalyse (gegen den echten Hermes-Code)

### 3.1 Tool-Zugriff aus der Sandbox — **[verifiziert: ja]**

Das war meine vermutete Hauptunsicherheit; der Code löst sie **positiv** auf:

- Hermes generiert pro Lauf ein **`hermes_tools.py`-Stub-Modul mit
  RPC-Funktionen**. Der vom Modell geschriebene Code macht
  `from hermes_tools import web_search, web_extract, read_file, write_file, ...`
  und ruft die Tools als normale Funktionen auf. **[verifiziert]**
- Der Code läuft als **Kindprozess auf dem Agent-Host** und kommuniziert mit
  Hermes über **Unix-Domain-Socket-RPC** (nur Linux/macOS). **[verifiziert]**

Wichtige Konsequenz, die meine erste Sorge entkräftet: **Netzwerkisolation des
Sandbox ist kein Hindernis.** Der Child-Prozess muss selbst gar nicht ins Netz
— `web_search`/`web_extract` werden per RPC vom **Host** ausgeführt. Genau das
ist der elegante Kern des Designs und der Grund, warum der Entwurf grundsätzlich
trägt.

Verfügbare Tools im Stub **[verifiziert]**: `web_search`, `web_extract`,
`read_file`, `write_file`, `search_files`, `patch`, `terminal` (nur
Vordergrund). **`memory` ist nicht dabei** — der `from hermes_tools import
memory`-Import des Entwurfs schlägt fehl.

### 3.2 Zustand zwischen `execute_code`-Läufen — **[verifiziert: zustandslos]**

„Hermes always writes the script and the auto-generated `hermes_tools.py` RPC
stub into a temp staging directory **that is cleaned up after execution**."
Jeder Aufruf ist ein **frischer Kindprozess** **[verifiziert]**.

Damit ist das Entwurfsmuster „Funktion in `execute_code`-Aufruf 1 definieren,
in Aufruf 2 als bereits geladen voraussetzen" **defekt**: Aufruf 2 wirft
`NameError` (und `json`/`time` fehlen ebenso, da nur in Snippet 1 importiert).
Der überarbeitete Entwurf lädt den Code pro Aufruf neu aus einer Datei/Skill
(5.4) und persistiert Zustand ausschließlich über `read_file`/`write_file`.

### 3.3 Tokenersparnis — **[verifiziert: real, by design]**

„**Intermediate tool results never enter the context window … only the final
`print()` output comes back, dramatically reducing token usage**"
**[verifiziert]**. Die großen Rohtreffer von `web_search`/`web_extract` bleiben
im Skript; nur das, was am Ende `print()`-ausgegeben wird, kostet Modell-Tokens.

Damit ist die Ersparnis-Behauptung des Entwurfs grundsätzlich **plausibel** —
mit zwei Präzisierungen:

- Nur **`print()`-Ausgabe** zählt, **nicht** der `return`-Wert der Funktion.
  Der Aufrufer muss das verdichtete Ergebnis explizit `print()`en (der Entwurf
  macht das im Beispiel korrekt; mein refaktoriertes `search_as_code` gibt einen
  Dict **zurück** und muss daher vom Aufrufer geprintet werden).
- Die konkrete Zahl „~85 %" ist Perplexitys Benchmark, nicht Hermes'. Real
  messen (5.8), aber die *Richtung* stimmt.

### 3.4 Sicherheit / Secrets — **[verifiziert]**

Der Entwurf behauptet „keine zusätzliche Isolation" — das stimmt für Hermes
**nicht**: Der Code-Child läuft mit **minimaler Umgebung**; Variablen mit
`KEY`, `TOKEN`, `SECRET`, `PASSWORD`, `CREDENTIAL`, `PASSWD`, `AUTH` werden
**standardmäßig entfernt**. API-Keys sind nur verfügbar, wenn sie per
**Skill-Frontmatter** oder in `config.yaml` **allowlisted** werden
**[verifiziert]**.

Praktische Folge: `web_search` braucht einen Web-API-Key, `web_extract` mit
`use_llm_processing=True` zusätzlich ein Auxiliary-Modell. Ohne Allowlisting
liefern beide Tools Fehler. Das gehört in das Skill-Frontmatter (5.4/5.7).

---

## 4. Schwächen im Detail

„Blocker" = verhindert korrekte Funktion auch bei korrekter Hermes-Anbindung.
„API" = falsche Annahme gegen den echten Hermes-Code.

| # | Schwere | Fundstelle | Problem | Fix |
|---|---------|-----------|---------|-----|
| 1 | **Blocker** | `is_relevant` | Verlangt das **gesamte** `base_query` als Teilstring in Titel/Snippet. Bei Mehrwort-Queries trifft das praktisch nie → **fast immer leeres Ergebnis**. | Token-Coverage mit Schwellwert (5.2). |
| 2 | **Blocker** | Rerank-Score | Belohnt **Textlänge** (`+= len(...)`) → bevorzugt geschwätzige Treffer; `content.count(base_query)` zählt die ganze Phrase → Beitrag ≈ 0. Faktisch Längen-Sortierung. | Längennormierter Term-Coverage-Score (5.2). |
| 3 | **Blocker** | `web_extract`-Mapping | Mappt `results[i] → raw_results[i]` **positionell**; bei Drops/Umsortierung landet Content am falschen Treffer. | Mapping per **URL** (`results[*].url` existiert) (5.5). |
| 4 | **API** | `from hermes_tools import memory` | `memory` ist **nicht** im RPC-Stub → ImportError. | `read_file`/`write_file` (5.3/5.5). |
| 5 | **API** | `web_extract(urls=urls[:10])` | Hermes erlaubt **max. 5 URLs**. | Auf 5 begrenzen (5.5). |
| 6 | **API** | Tool-Rückgaben | `web_search`/`web_extract` liefern **JSON-Strings**, nicht Dicts; `.get(...)` direkt darauf scheitert. | `json.loads`, wenn `str` (5.5). |
| 7 | **API** | „keine Isolation" + Keys | Child hat **minimale Umgebung ohne Secrets**; Keys müssen allowlisted werden. | Skill-Frontmatter/`config.yaml` (5.4/5.7). |
| 8 | Hoch | Zustand | Verlässt sich auf REPL-Persistenz; Hermes ist **zustandslos**. | Pro-Aufruf-Laden + Datei-State (5.4). |
| 9 | Hoch | Pipeline-Reihenfolge | Extraktion läuft **vor** Filter/Dedupe → teure `web_extract`-Calls für später verworfene Treffer. | Dedupe → Filter → Rerank → **nur Top-K extrahieren** (5.1). |
| 10 | Hoch | Fanout | „Fanout" = `site:de/org/edu`. `site:org` ist kein gültiger TLD-Filter und `.com/.net` wird verworfen. Quellen-Restriktion statt semantischem Fanout. | Echte Query-Varianten; TLD nur optional (5.2). |
| 11 | Mittel | URL-Dedupe | `uid = url or title` ohne **Normalisierung** → `http/https`, Trailing-Slash, `utm_*` erzeugen Dubletten. | URL-Normalisierung (5.5). |
| 12 | Mittel | Cache-Pfad | Schreibt nach `~`-Pfad ohne Verzeichnis-Anlage; Staging-Dir wird ohnehin gelöscht. | Absoluter, **persistenter** Pfad außerhalb des Staging-Dirs (5.3). |
| 13 | Mittel | `print`-Warnungen | Landen im LLM-Output → blähen Context, untergraben die Tokenersparnis. | Strukturiertes `log`-Feld (5.5/5.8). |
| 14 | Niedrig | Cron/Modell-ID | YAML-Schema **[zu prüfen]**; Modell `nvidia/nemotron-3-super-120b-a12b:free` wirkt verstümmelt. | Gegen Hermes-Cron-Doku prüfen (6). |
| 15 | Niedrig | „parallel" | Kommentar sagt parallel, Code ist sequentiell → Latenz ∝ Quellen × Extraktionen. | Optional via Subagent/`delegate_task` (5.6). |

---

## 5. Überarbeiteter Designentwurf

### 5.1 Architektur & Datenfluss

Hermes implementiert das SaC-Muster nativ (3.1/3.3): **ein** `execute_code`-Lauf
orchestriert alles, Tool-Ergebnisse bleiben im Skript, nur das verdichtete
Endergebnis wird ge`print()`et. Korrigierte Pipeline:

```
fanout(base_query) → [sub_queries]
   → retrieve (web_search je sub_query)        # JSON-String → dict
   → merge + URL-Normalisierung
   → dedupe (nach normalisierter URL)          # früh, spart Arbeit
   → filter (Token-Coverage ≥ Schwelle)        # vor teurer Extraktion
   → rerank (längennormiert, Term-Coverage)
   → select Top-K
   → extract NUR Top-K, max. 5 (web_extract)   # teuer, daher minimal
   → optional: re-rank mit Volltext-Signal
   → cache (zweischichtig, write_file) + print(verdichtetes Ergebnis)
```

Kernänderung gegenüber dem Entwurf: **Extraktion ganz nach hinten**, nur für die
gefilterten Top-K (≤ 5 wegen Hermes-Limit) — der größte Kostenhebel.

### 5.2 Pipeline-Stufen (korrigiert)

- **Fanout:** deterministische Query-Varianten aus einer übergebenen Liste
  (Synonyme/Reformulierungen); TLD/`site:` nur als *optionaler* Zusatz, nie als
  Ersatz für `.com` & Co. Varianten optional einmalig modellgeneriert (5.8).
- **Filter:** Query in Terme zerlegen; Treffer passt, wenn **Term-Abdeckung** in
  Titel+Snippet ≥ Schwellwert (z. B. 0,5). Kein Ganz-Phrasen-Substring.
- **Rerank:** transparenter, **längennormierter** Score: Term-Coverage
  (Hauptsignal) + normierte Term-Frequenz + Titel-Bonus. Hook für späteren
  Embedding-Rerank, falls verfügbar **[zu prüfen]**.

### 5.3 Caching (zweischichtig, über `read_file`/`write_file`)

Da `memory` im Stub fehlt und das Staging-Verzeichnis gelöscht wird, läuft der
Cache über **`read_file`/`write_file` auf einen persistenten, absoluten Pfad**
(z. B. ein konfiguriertes Hermes-Datenverzeichnis; FS-Policy beachten,
**[zu prüfen]**).

1. **Query-Ergebnis-Cache** — Key aus normalisierter Query-Menge **und allen
   ergebnisrelevanten Parametern**. TTL pro Thema (zeitkritisch 30 min, statisch
   Stunden).
2. **URL-Content-Cache** — Key = normalisierte URL → teure `web_extract`-Ergebnisse
   werden **über Queries hinweg** wiederverwendet (großer Hebel beim Monitoring).

### 5.4 Zustands-/Skill-Modell (zustandssicher)

Hermes ist zustandslos (3.2), daher:

- `search_as_code` **als Datei/Skill** ablegen, nicht inline in jedem Aufruf.
- Jeder `execute_code`-Aufruf lädt sie neu (Skill-Mechanismus bzw. `read_file`
  + `exec`/Import aus einem persistenten Pfad **[zu prüfen]**, ob ein Skill
  importierbaren Code bereitstellen kann oder nur Text liefert).
- **Secrets** für `web_search`/`web_extract` im **Skill-Frontmatter** bzw.
  `config.yaml` allowlisten (3.4) — sonst Tool-Fehler.

### 5.5 Referenzimplementierung (korrigiert, gegen echte Hermes-API)

> Anpassungen ggü. Entwurf: `read_file`/`write_file` statt `memory`;
> `json.loads` für JSON-String-Rückgaben; `web_extract` auf 5 URLs begrenzt;
> Extraktion zuletzt und per URL gemappt; Token-Coverage-Filter;
> längennormiertes Rerank; Warnungen ins `log`-Feld. Exakte Signaturen von
> `read_file`/`write_file` ggf. anpassen **[zu prüfen]**.

```python
"""search_as_code.py — Search-as-Code-Orchestrierung für Hermes.

Pipeline: fanout -> retrieve -> normalize -> dedupe -> filter -> rerank
          -> Top-K -> extract (nur Top-K, max 5) -> (re-rank) -> cache.
Hinweis: gibt einen Dict ZURÜCK; der Aufrufer muss ihn print()en, damit
nur das verdichtete Ergebnis ins Context-Window gelangt.
"""
from __future__ import annotations
import hashlib, json, os, re, time
from urllib.parse import urlsplit, urlunsplit, parse_qsl, urlencode

from hermes_tools import web_search, web_extract, read_file, write_file

# Persistenter Pfad AUSSERHALB des (gelöschten) Staging-Dirs; FS-Policy beachten.
CACHE_DIR = "/opt/hermes/data/search_cache"
CONTENT_DIR = "/opt/hermes/data/content_cache"
_TRACKING = re.compile(r"^(utm_|fbclid$|gclid$|mc_eid$|ref$)")


# ---------- Hilfen ---------------------------------------------------------
def _as_dict(resp) -> dict:
    """web_search/web_extract liefern JSON-Strings."""
    if isinstance(resp, str):
        try:
            return json.loads(resp)
        except Exception:
            return {}
    return resp or {}


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
    return sum(1 for qt in query_terms if qt in blob) / len(query_terms)


def _read_json(path: str) -> dict | None:
    try:
        raw = read_file(path=path)               # Signatur ggf. anpassen
        text = raw if isinstance(raw, str) else (raw or {}).get("content", "")
        return json.loads(text) if text else None
    except Exception:
        return None


def _write_json(path: str, obj: dict) -> None:
    try:
        os.makedirs(os.path.dirname(path), exist_ok=True)
        write_file(path=path, content=json.dumps(obj, ensure_ascii=False))
    except Exception:
        pass  # Cache ist best-effort, nie Fehlerquelle


# ---------- Stufen ---------------------------------------------------------
def fanout(base_query, variants, tld_sites) -> list[str]:
    queries = [base_query] + [f"{base_query} {v}" for v in (variants or [])]
    if tld_sites:                                # optionaler Zusatz, kein Ersatz
        queries += [f"{base_query} site:{d}" for d in tld_sites]
    return list(dict.fromkeys(queries))          # Reihenfolge erhalten + dedupe


def rerank_score(query_terms: set[str], r: dict) -> float:
    title, snippet, content = r.get("title", ""), r.get("snippet", ""), r.get("content", "")
    cov = _coverage(query_terms, title, snippet, content)        # Hauptsignal
    blob = _terms(f"{title} {snippet} {content}")
    tf = (sum(t in query_terms for t in blob) / (len(blob) + 1)) if blob else 0.0
    return 3.0 * cov + 1.0 * tf + 0.5 * _coverage(query_terms, title)


# ---------- Hauptfunktion --------------------------------------------------
def search_as_code(
    base_query: str,
    query_variants: list[str] | None = None,
    tld_sites: list[str] | None = None,
    max_results_per_query: int = 5,
    top_k: int = 10,
    extract_top_k: int = 5,          # Hermes web_extract: HARTES Limit = 5
    min_coverage: float = 0.5,
    cache_ttl_sec: int = 3600,
) -> dict:
    log: list[str] = []
    qterms = set(_terms(base_query))

    # 0) Query-Ergebnis-Cache
    ckey = hashlib.sha256(json.dumps({
        "q": base_query, "v": query_variants, "t": tld_sites,
        "n": max_results_per_query, "k": top_k, "e": extract_top_k,
        "mc": min_coverage,
    }, sort_keys=True).encode()).hexdigest()
    cpath = f"{CACHE_DIR}/{ckey}.json"
    cached = _read_json(cpath)
    if cached and time.time() - cached.get("_ts", 0) < cache_ttl_sec:
        return {"results": cached["results"], "cached": True, "log": []}

    # 1) Fanout + 2) Retrieve
    raw: list[dict] = []
    for sq in fanout(base_query, query_variants, tld_sites):
        try:
            data = _as_dict(web_search(query=sq, limit=max_results_per_query))
            for r in data.get("data", {}).get("web", []):
                raw.append({"title": r.get("title", ""),
                            "url": r.get("url", ""),
                            "snippet": r.get("description", ""),
                            "source": sq})
        except Exception as e:
            log.append(f"web_search fehlgeschlagen für {sq!r}: {e}")

    # 3) Normalisieren + 4) Dedupe (nach normalisierter URL)
    seen, items = set(), []
    for r in raw:
        nu = normalize_url(r["url"])
        key = nu or r["title"]
        if key and key not in seen:
            seen.add(key)
            r["norm_url"] = nu
            items.append(r)

    # 5) Filter (Token-Coverage)
    items = [r for r in items
             if _coverage(qterms, r["title"], r["snippet"]) >= min_coverage]

    # 6) Rerank (vor Extraktion) + 7) Top-K
    for r in items:
        r["score"] = rerank_score(qterms, r)
    items.sort(key=lambda x: x["score"], reverse=True)
    items = items[:top_k]

    # 8) Extraktion NUR der Top-K, max. 5, Mapping per URL
    to_extract = [r for r in items[:extract_top_k] if r["norm_url"]][:5]
    if to_extract:
        by_url, missing = {}, []
        for r in to_extract:                       # URL-Content-Cache zuerst
            h = hashlib.sha256(r["norm_url"].encode()).hexdigest()
            hit = _read_json(f"{CONTENT_DIR}/{h}.json")
            if hit and time.time() - hit.get("_ts", 0) < cache_ttl_sec:
                by_url[r["norm_url"]] = hit["content"]
            else:
                missing.append(r["norm_url"])
        if missing:
            try:
                ext = _as_dict(web_extract(urls=missing[:5]))  # HARTES Limit 5
                for e in ext.get("results", []):
                    if e.get("error"):
                        continue
                    nu = normalize_url(e.get("url", ""))
                    by_url[nu] = e.get("content", "")
                    h = hashlib.sha256(nu.encode()).hexdigest()
                    _write_json(f"{CONTENT_DIR}/{h}.json",
                                {"_ts": time.time(), "content": by_url[nu]})
            except Exception as e:
                log.append(f"web_extract fehlgeschlagen: {e}")
        for r in items:
            if r["norm_url"] in by_url:
                r["content"] = by_url[r["norm_url"]]
        for r in items:                            # 9) Re-Rank mit Volltext
            r["score"] = rerank_score(qterms, r)
        items.sort(key=lambda x: x["score"], reverse=True)

    # 10) Schlanke Rückgabe + Cache
    slim = [{"title": r["title"], "url": r["norm_url"] or r["url"],
             "snippet": r["snippet"], "score": round(r["score"], 3)}
            for r in items]
    _write_json(cpath, {"_ts": time.time(), "results": slim})
    return {"results": slim, "cached": False, "log": log}
```

Aufruf (nur das Ergebnis landet im Context-Window):

```python
out = search_as_code("Perplexity Search as Code Tokenersparnis",
                     query_variants=["Benchmark", "vs klassische Pipeline"])
print(json.dumps(out["results"], ensure_ascii=False))   # NUR print() zählt
```

### 5.6 Scheduling / Monitoring

- Cron ruft ein Skill auf, das `search_as_code(...)` lädt und das Ergebnis an
  einen kurzen LLM-Zusammenfassungsschritt gibt → wenige Tokens.
- TTL pro Thema steuern.
- Optionale Parallelisierung großer Fanouts über Subagenten — nur bei
  nachgewiesenem Latenzbedarf.
- Cron-YAML-Schema, `deliver`-Kanäle und Modell-IDs **[zu prüfen]** gegen die
  Hermes-Doku; die Original-Modell-ID wirkt verstümmelt — nicht ungeprüft
  übernehmen.

### 5.7 Sicherheit — angepasst an Hermes' echtes Modell

- Hermes' Code-Child hat **minimale Umgebung ohne Secrets** (3.4) — das ist ein
  Vorteil, kein „keine Isolation". Für `web_search`/`web_extract` benötigte Keys
  **explizit** per Skill-Frontmatter/`config.yaml` allowlisten; nichts darüber
  hinaus freigeben.
- Skill-Code **versionieren und reviewen**; keinen ungeprüften Ad-hoc-Code
  ausführen.
- `web_extract`-Inhalte sind **nicht vertrauenswürdig** — nur als Daten
  behandeln, nie als Anweisungen (Prompt-Injection im Zusammenfassungsschritt).
  Beachten: `web_extract` führt mit `use_llm_processing=True` bereits eine
  LLM-Verarbeitung des Fremdinhalts durch.

### 5.8 Observability & ehrliche Kostenrechnung

- **Tokenersparnis** ist by design real (3.3): nur `print()`-Ausgabe zählt.
  Trotzdem vorher/nachher mit echten Token-Zählern messen statt „85 %" zu
  übernehmen.
- `web_extract` mit `use_llm_processing=True` kostet ein **Auxiliary-Modell** je
  Lauf — bei reinem Snippet-Bedarf abschalten (`use_llm_processing=False`).
- **Logs** ins `log`-Feld, nicht nach stdout, um den Context nicht zu fluten.

---

## 6. Verifikations-Checkliste

Erledigt durch Code-Recherche:

- [x] In-Sandbox-Toolzugriff (`hermes_tools`-RPC-Stub) — **ja** (3.1).
- [x] Tool-Ergebnisse umgehen das Context-Window (`print()`-only) — **ja** (3.3).
- [x] Zustand zwischen Läufen — **zustandslos** (3.2).
- [x] `web_search`-Rückgabe `data.web[*]{title,url,description}` (JSON-String) (3.x).
- [x] `web_extract`: **max. 5 URLs**, `results[*]{url,title,content,error}` (JSON-String).
- [x] Secret-Handling: Keys standardmäßig entfernt, Allowlist nötig (3.4).

Noch zu prüfen vor dem Bau:

- [ ] Exakte Signatur/Rückgabe von `read_file`/`write_file` im Stub.
- [ ] Persistenter, beschreibbarer Pfad unter der FS-Policy (außerhalb Staging).
- [ ] Ob der RPC-Stub JSON-Strings oder bereits geparste Dicts liefert
      (der Code behandelt beide Fälle).
- [ ] Skill-System: kann ein Skill **importierbaren Code** bereitstellen, und
      wie wird der Web-API-Key im Frontmatter allowlisted?
- [ ] Cron-System: YAML-Schema, `deliver`-Kanäle, gültige Modell-IDs.
- [ ] Embeddings für optionalen Rerank-Upgrade verfügbar?

---

## 7. Umsetzungs-Roadmap

1. **Rest-Verifikation** (Abschnitt 6, offene Punkte — v. a. Pfad + `read_file`/
   `write_file`-Signatur + Key-Allowlisting).
2. **MVP**: `search_as_code` mit korrigiertem Filter/Rerank/Mapping und
   einschichtigem Query-Cache; an *einer* realen Query Treffergüte und Tokens
   (vorher/nachher) messen.
3. **Cache-Layer 2** (URL-Content-Cache) + Skill-Verpackung inkl. Key-Allowlist.
4. **Scheduling** erst nach bestätigtem Cron-Schema; kurzer
   Zusammenfassungsschritt + Zustellkanal.
5. **Optimierung**: optionaler Embedding-Rerank, Subagent-Fanout nur bei
   nachgewiesenem Latenzbedarf.

---

## 8. Fazit

Gegen den echten Hermes-Code geprüft, ist der Entwurf **konzeptionell goldrichtig**:
Hermes' „Programmatic Tool Calling" *ist* Search as Code, inklusive der
Tokenersparnis durch `print()`-only-Rückgabe. Meine anfängliche Hauptsorge
(In-Sandbox-Toolzugriff) ist **positiv aufgelöst** — der RPC-Stub macht sie
gegenstandslos, und Netzisolation ist dank Host-RPC kein Problem.

Lauffähig ist der konkrete Code dennoch erst nach Behebung von drei
Logik-Blockern (Filter, Rerank, Extract-Mapping) und vier echten
API-Fehlannahmen (`memory` existiert nicht → `read_file`/`write_file`;
`web_extract`-Limit 5; JSON-String-Rückgaben; Secrets standardmäßig entfernt)
sowie der Umstellung auf das **zustandslose** Ausführungsmodell. Abschnitt 5
liefert die korrigierte, an die echte Hermes-API angepasste Implementierung;
die wenigen verbleibenden Unbekannten sind in Abschnitt 6 als prüfbare Punkte
isoliert.
