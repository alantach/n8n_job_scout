# Prompt: Vytvoř Job Scout workflow

Vytvoř systém pro automatické scrapování a prescreening pracovních inzerátů. Systém se skládá ze dvou variant: **Python skript** (pro GitHub Actions nebo lokální spuštění) a **n8n workflow** (pro self-hosted n8n v Dockeru).

---

## Co systém dělá

1. Načte seznam URL zdrojů z `data/urls.json`
2. Načte hodnotící pravidla z `data/pravidla.json`
3. Pro každý aktivní zdroj stáhne HTML výpis inzerátů
4. Extrahuje odkazy na jednotlivé inzeráty podle vzoru `url_vzor`
5. Stáhne detail každého inzerátu
6. Parsuje název pozice, firmu, lokalitu a popis
7. Spočítá prescoring podle pravidel (součet vah matching klíčových slov)
8. Inzeráty pod minimálním skóre zahodí
9. Zkontroluje duplicity podle URL odkazu
10. Nové inzeráty uloží do `data/vysledky.json`

---

## Struktura souborů

```
projekt/
  scout.py                          ← hlavní Python skript
  data/
    urls.json                       ← seznam zdrojů ke scrapování
    pravidla.json                   ← hodnotící pravidla s váhami
    vysledky.json                   ← výsledky (prázdné [] na začátku)
  .github/
    workflows/
      scout.yml                     ← GitHub Actions workflow (volitelné)
```

---

## Konfigurační soubory

### `data/urls.json`

Seznam zdrojů ke scrapování. Každý zdroj je objekt s těmito poli:

| Pole | Typ | Popis |
|------|-----|-------|
| `aktivni` | boolean | `true` = zdroj se scrapuje, `false` = přeskočí se |
| `nazev` | string | Lidsky čitelný název zdroje (zobrazuje se v logu) |
| `url` | string | URL stránky s výpisem inzerátů |
| `url_vzor` | string | Část URL která musí být v odkazu na detail inzerátu |

**Příklad:**
```json
[
  {
    "aktivni": true,
    "nazev": "Jobs.cz - IT analytik",
    "url": "https://www.jobs.cz/prace/?field[]=200900012",
    "url_vzor": "/rpd/"
  },
  {
    "aktivni": true,
    "nazev": "Jobstack IT",
    "url": "https://www.jobstack.it/it-jobs",
    "url_vzor": "/it-jobs/"
  },
  {
    "aktivni": false,
    "nazev": "Vypnutý zdroj",
    "url": "https://example.com/jobs",
    "url_vzor": "/job/"
  }
]
```

**Jak funguje `url_vzor`:** Skript najde všechny `href` odkazy na stránce výpisu. Zachová pouze ty, které v URL obsahují řetězec `url_vzor` (case-insensitive). Tím se odfiltrují navigační odkazy, reklamy atd. a zůstanou jen skutečné inzeráty.

---

### `data/pravidla.json`

Pravidla pro prescoring. Každé pravidlo definuje klíčové slovo a váhu. Skript hledá klíčové slovo (case-insensitive) v textu složeném z názvu pozice + firma + lokalita + popis inzerátu.

| Pole | Typ | Popis |
|------|-----|-------|
| `kategorie` | string | Jen informativní štítek (TECHNOLOGIE, POZICE, LOKALITA, MINUS) |
| `pravidlo` | string | Klíčové slovo nebo fráze k hledání v textu inzerátu |
| `vaha` | integer | Kladná = přidá body, záporná = ubere body |
| `popis` | string | Lidský popis pravidla (jen pro dokumentaci) |

**Výsledné skóre** = součet všech matchujících vah, oříznutý na rozsah 1–10.

**Minimální skóre** pro uložení je ve skriptu nastaveno konstantou `MIN_SKORE = 5`. Inzeráty pod tímto prahem se zahodí.

**Příklad:**
```json
[
  { "kategorie": "TECHNOLOGIE", "pravidlo": "IBM webMethods", "vaha": 5,  "popis": "Klíčová technologie" },
  { "kategorie": "TECHNOLOGIE", "pravidlo": "MuleSoft",       "vaha": 4,  "popis": "Alternativní platforma" },
  { "kategorie": "TECHNOLOGIE", "pravidlo": "ESB",            "vaha": 4,  "popis": "Enterprise Service Bus" },
  { "kategorie": "TECHNOLOGIE", "pravidlo": "REST API",       "vaha": 3,  "popis": "API znalost" },
  { "kategorie": "POZICE",      "pravidlo": "solution architect",    "vaha": 5,  "popis": "Hledaná pozice" },
  { "kategorie": "POZICE",      "pravidlo": "integration architect", "vaha": 5,  "popis": "Hledaná pozice" },
  { "kategorie": "POZICE",      "pravidlo": "IT analytik",           "vaha": 4,  "popis": "Hledaná pozice" },
  { "kategorie": "LOKALITA",    "pravidlo": "remote",   "vaha": 3,  "popis": "Vzdálená práce" },
  { "kategorie": "LOKALITA",    "pravidlo": "Praha",    "vaha": 2,  "popis": "Preferovaná lokalita" },
  { "kategorie": "MINUS",       "pravidlo": "junior",   "vaha": -3, "popis": "Nechceme juniorní pozice" },
  { "kategorie": "MINUS",       "pravidlo": "stáž",     "vaha": -5, "popis": "Stáže vyloučit" },
  { "kategorie": "MINUS",       "pravidlo": "helpdesk", "vaha": -4, "popis": "Helpdesk nerelevantní" }
]
```

---

### `data/vysledky.json`

Výstupní soubor. Na začátku prázdné pole `[]`. Skript při každém běhu přidá nové záznamy a soubor přepíše. Záznamy jsou seřazeny sestupně podle `prescoring`.

**Struktura jednoho záznamu:**
```json
{
  "datum":          "2026-03-16 08:23:11",
  "zdroj_nazev":    "Jobs.cz - IT analytik",
  "nazev_pozice":   "Solution Architect - integrace",
  "firma":          "Acme Corp s.r.o.",
  "lokalita":       "Praha",
  "odkaz":          "https://www.jobs.cz/rpd/abc123/",
  "prescoring":     8,
  "match_pravidla": "solution architect(+5), REST API(+3), remote(+3), junior(-3)"
}
```

---

## Python skript `scout.py`

Skript nepoužívá žádné externí knihovny — pouze standardní Python 3 (`urllib`, `json`, `re`, `os`, `datetime`).

**Vygeneruj tento skript:**

```python
#!/usr/bin/env python3
"""
Job Scout - prescreening
Vysledky ulozi do data/vysledky.json
"""
import urllib.request
import json
import os
import re
from datetime import datetime

BASE_DIR   = os.path.dirname(os.path.abspath(__file__))
DATA_DIR   = os.path.join(BASE_DIR, "data")
URLS_FILE  = os.path.join(DATA_DIR, "urls.json")
RULES_FILE = os.path.join(DATA_DIR, "pravidla.json")
OUT_FILE   = os.path.join(DATA_DIR, "vysledky.json")
MIN_SKORE  = 5

HEADERS = {
    "User-Agent":      "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36",
    "Accept-Language": "cs-CZ,cs;q=0.9",
    "Accept":          "text/html,application/xhtml+xml,*/*;q=0.8",
}

def fetch(url):
    try:
        req = urllib.request.Request(url, headers=HEADERS)
        with urllib.request.urlopen(req, timeout=20) as r:
            return r.read().decode("utf-8", errors="ignore")
    except Exception as e:
        print(f"    ! {e}")
        return ""

def extrahuj_odkazy(html, base_url, vzor):
    links = []
    origin = re.match(r'^(https?://[^/]+)', base_url)
    origin = origin.group(1) if origin else ""
    for m in re.finditer(r'href=["\']([^"\'#?][^"\']*)["\']', html):
        href = m.group(1)
        if href.startswith("http"):
            full = href
        elif href.startswith("/"):
            full = origin + href
        else:
            continue
        if vzor and vzor.lower() not in full.lower():
            continue
        if full not in links:
            links.append(full)
    return links

def parsuj(html, odkaz):
    text = re.sub(r'<script[\s\S]*?</script>', ' ', html, flags=re.I)
    text = re.sub(r'<style[\s\S]*?</style>', ' ', text, flags=re.I)
    text = re.sub(r'<[^>]+>', ' ', text)
    text = re.sub(r'&\w+;', ' ', text)
    text = re.sub(r'\s+', ' ', text).strip()

    nazev = ""
    m = re.search(r'<h1[^>]*>([\s\S]{1,200}?)</h1>', html, re.I)
    if m: nazev = re.sub(r'<[^>]+>', '', m.group(1)).strip()
    if not nazev:
        m = re.search(r'og:title[^>]*content=["\']([^"\']+)["\']', html, re.I)
        if m: nazev = m.group(1).strip()
    if not nazev:
        m = re.search(r'<title[^>]*>([^<]+)</title>', html, re.I)
        if m: nazev = re.split(r'[|\-–]', m.group(1))[0].strip()

    firma = ""
    m = re.search(r'hiringOrganization[\s\S]{0,300}?"name"\s*:\s*"([^"]{1,100})"', html, re.I)
    if m: firma = m.group(1)
    if not firma:
        m = re.search(r'og:site_name[^>]*content=["\']([^"\']+)["\']', html, re.I)
        if m: firma = m.group(1).strip()

    lokalita = ""
    m = re.search(r'addressLocality["\':\s]+([^\s"\'}{,<]{2,50})', html, re.I)
    if m: lokalita = m.group(1).strip()

    return {
        "nazev_pozice": nazev[:150],
        "firma":        firma[:100],
        "lokalita":     lokalita[:100],
        "popis":        text[:2000],
        "odkaz":        odkaz,
    }

def prescoring(inzerat, rules):
    haystack = " ".join([
        inzerat["nazev_pozice"],
        inzerat["firma"],
        inzerat["lokalita"],
        inzerat["popis"],
    ]).lower()
    skore, matched = 0, []
    for r in rules:
        if r["pravidlo"].lower() in haystack:
            skore += r.get("vaha", 0)
            matched.append(f"{r['pravidlo']}({'+' if r['vaha']>0 else ''}{r['vaha']})")
    return max(1, min(10, skore)), ", ".join(matched)

def main():
    with open(URLS_FILE,  "r", encoding="utf-8") as f: urls  = json.load(f)
    with open(RULES_FILE, "r", encoding="utf-8") as f: rules = json.load(f)

    if os.path.exists(OUT_FILE):
        with open(OUT_FILE, "r", encoding="utf-8") as f:
            vysledky = json.load(f)
    else:
        vysledky = []

    existujici = {v["odkaz"] for v in vysledky}
    aktivni    = [u for u in urls if u.get("aktivni")]
    nove       = 0
    preskoceno = 0

    print(f"\n=== Job Scout {datetime.now().strftime('%Y-%m-%d %H:%M')} ===")
    print(f"Zdroju: {len(aktivni)}, pravidel: {len(rules)}, existujicich: {len(vysledky)}\n")

    for zdroj in aktivni:
        print(f"[{zdroj['nazev']}]")
        html_vypis = fetch(zdroj["url"])
        if not html_vypis:
            print("  PRESKOCENO\n")
            continue

        odkazy = extrahuj_odkazy(html_vypis, zdroj["url"], zdroj.get("url_vzor", ""))
        print(f"  Odkazu: {len(odkazy)}")

        for odkaz in odkazy:
            if odkaz in existujici:
                continue
            html_detail = fetch(odkaz)
            if not html_detail:
                continue

            inzerat = parsuj(html_detail, odkaz)
            skore, matched = prescoring(inzerat, rules)

            if skore < MIN_SKORE:
                preskoceno += 1
                continue

            vysledky.append({
                "datum":          datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
                "zdroj_nazev":    zdroj["nazev"],
                "nazev_pozice":   inzerat["nazev_pozice"],
                "firma":          inzerat["firma"],
                "lokalita":       inzerat["lokalita"],
                "odkaz":          odkaz,
                "prescoring":     skore,
                "match_pravidla": matched,
            })
            existujici.add(odkaz)
            nove += 1
            print(f"  + [{skore}/10] {inzerat['nazev_pozice'][:60]} @ {inzerat['firma'][:30]}")

        print()

    vysledky.sort(key=lambda x: x["prescoring"], reverse=True)

    with open(OUT_FILE, "w", encoding="utf-8") as f:
        json.dump(vysledky, f, ensure_ascii=False, indent=2)

    print(f"=== Hotovo: {nove} novych, {preskoceno} pod skore {MIN_SKORE}, celkem: {len(vysledky)} ===\n")

if __name__ == "__main__":
    main()
```

---

## GitHub Actions workflow `.github/workflows/scout.yml`

Spouští `scout.py` každý všední den v 6:00 UTC a commituje výsledky zpět do repozitáře. Nevyžaduje žádné secrets — skript nepoužívá žádné API.

```yaml
name: Job Scout

on:
  schedule:
    - cron: '0 6 * * 1-5'   # Po-Pá v 6:00 UTC = 7:00 nebo 8:00 CZ
  workflow_dispatch:          # Ruční spuštění z GitHubu

jobs:
  scout:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Spust scout
        run: python scout.py

      - name: Commitni vysledky
        run: |
          git config user.name  "Job Scout Bot"
          git config user.email "scout@github-actions"
          git add data/vysledky.json
          git diff --cached --quiet || git commit -m "Scout $(date +'%Y-%m-%d') - nove nabidky"
          git push
```

---

## n8n workflow (bez AI, bez validace duplicit)

Pro self-hosted n8n v Dockeru. Vygeneruj Python skript `create_workflow2_scout.py` který přes n8n API vytvoří workflow.

**Spuštění:**
```cmd
python create_workflow2_scout.py TVUJ_N8N_API_KLIC
```

### Architektura workflow

```
Manualni spusteni
  → Nacti URLs (Code - čte /data/urls.json přes fs)
  → Filtruj aktivni URL (Code - filtruje aktivni:true)
  → Loop pres URL (splitInBatches po 1)
      → Nacti pravidla (Code - čte /data/pravidla.json přes fs)
      → Spoj pravidla s URL (Code - přidá _rules do URL itemu)
      → Stahni vypis (HTTP GET na url zdroje)
      → Extrahuj odkazy (Code - regex href, filtr url_vzor)
      → Loop pres inzerat (splitInBatches po 1)
          → Stahni detail (HTTP GET na odkaz inzerátu)
          → Parsuj detail (Code - extrahuje nazev/firma/lokalita/popis)
          → Pre-scoring (Code - součet vah z pravidel)
          → Filtr prescoring 5 (Filter - prescoring >= 5)
          → Zkontroluj a uloz (Code - dedup + zápis do /data/vysledky.json)
          → zpět na Loop pres inzerat
      → zpět na Loop pres URL
```

### Popis každého nodu

**Nacti URLs** — Code node, čte soubor přes Node.js `fs`:
```javascript
const fs = require('fs');
const urls = JSON.parse(fs.readFileSync('/data/urls.json', 'utf8'));
return urls.map(function(u){ return { json: u }; });
```
Vyžaduje v docker-compose.yml: `NODE_FUNCTION_ALLOW_BUILTIN=fs`

**Filtruj aktivni URL** — Code node, vrátí jen záznamy kde `aktivni === true`.

**Loop pres URL** — splitInBatches, batchSize: 1. Output 0 = done (nikam nevede), Output 1 = každá URL.

**Nacti pravidla** — Code node, čte `pravidla.json` přes `fs`.

**Spoj pravidla s URL** — Code node, zkombinuje data z Loop pres URL s načtenými pravidly do jednoho itemu s polem `_rules`.

**Stahni vypis** — HTTP GET, URL: `={{ $('Loop pres URL').item.json.url }}`, timeout 30s, header User-Agent.

**Extrahuj odkazy** — Code node. Regex přes všechny `href` atributy, sestaví absolutní URL, filtruje podle `url_vzor`. Vrátí pole itemů s polem `odkaz_inzeratu`.

**Loop pres inzerat** — splitInBatches, batchSize: 1. Output 0 = done → zpět na Loop pres URL, Output 1 = každý inzerát.

**Stahni detail** — HTTP GET, URL: `={{ $json.odkaz_inzeratu }}`, timeout 30s.

**Parsuj detail** — Code node. Odstraní `<script>`, `<style>`, HTML tagy. Extrahuje:
- `nazev_pozice`: z `<h1>`, pak `og:title`, pak `<title>`
- `firma`: z JSON-LD `hiringOrganization`, pak `og:site_name`
- `lokalita`: z JSON-LD `addressLocality`
- `popis`: čistý text, max 2000 znaků

**Pre-scoring** — Code node. Sestaví haystack (nazev+firma+lokalita+popis lowercase). Pro každé pravidlo z `_rules` zkontroluje přítomnost klíčového slova, přičte váhu. Výsledek ořízne na 1–10.

**Filtr prescoring 5** — Filter node. Podmínka: `prescoring >= 5`. Output 0 = prošlo, Output 1 = nepovleno → zpět na Loop pres inzerat.

**Zkontroluj a uloz** — Code node. Načte `vysledky.json` přes `fs`, zkontroluje zda `odkaz_inzeratu` už existuje, pokud ne přidá nový záznam a přepíše soubor. Data čte z nodu `Pre-scoring` (`$('Pre-scoring').item.json`).

### Důležité poznámky k n8n

- `fs` modul musí být povolen: `NODE_FUNCTION_ALLOW_BUILTIN=fs` v docker-compose environment
- Soubory jsou v `/data/` uvnitř kontejneru (mapováno z `./data/` na hostu)
- n8n si interně používá port 5679 pro Task Broker — nepoužívej tento port pro vlastní služby
- Loop pres URL Output 0 (done) nesmí nikam vést — jinak vznikne nekonečná smyčka

---

## Přizpůsobení pro jiného uživatele

Změn jsou potřeba jen v konfiguračních souborech:

1. **`data/urls.json`** — nahraď URL zdroji relevantními pro tvůj obor a lokalitu
2. **`data/pravidla.json`** — nastav klíčová slova podle hledané pozice a technologií
3. **`MIN_SKORE`** v `scout.py` — uprav minimální práh (default 5)
4. **Cron čas** v `scout.yml` — uprav podle časového pásma (`0 6 * * 1-5` = 6:00 UTC)

Skript ani workflow nevyžadují žádné API klíče, databáze ani externí služby.
