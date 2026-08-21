# Knowledge Explorer — czat web z dostępem do bazy wiedzy NASK

> Feature spec (business-feature). Status: **DoR-ready**, publikacja na polecenie użytkownika.

## Tickets

Inicjatywa podzielona na 4 tickety (dzieci parenta #1):

- **#2 — Baza rozwiązania (pi agent as-is)** — ten spec. Scaffold z blueprintu `web-agentic`,
  agent pi **bez wycinania** filesystemu/worktree itp., + custom tool `rag_search`
  + system prompt czatu klienckiego.
- **#4 — Rozszerzenia funkcjonalne** — Deferred. Lead on (mail, kontakt z człowiekiem,
  dzwonienie), ew. inne rozszerzenia.
- **#5 — Minimalizacja** — Deferred. Usunięcie zbędnych elementów (filesystem, worktree,
  plugins/skills, bash, selektor modeli) + konstrukcyjne egzekwowanie read-only.
- **#3 — Microsandbox i bezpieczeństwo** — Deferred. Dockerfile, microsandbox, izolacja
  sesji/środowiska, hardening.

- Backlog parent: https://github.com/michal-michaluk/nask/issues/1
- Spec ticket (baza): https://github.com/michal-michaluk/nask/issues/2
- Deferred: https://github.com/michal-michaluk/nask/issues/3 · https://github.com/michal-michaluk/nask/issues/4 · https://github.com/michal-michaluk/nask/issues/5

## Business Context

### Intro

Osobna apka webowa — czat, w którym klient NASK rozmawia po polsku z asystentem mającym
dostęp do istniejącej bazy wiedzy `nask`. Czat odpowiada na pytania o produkty, oferty,
firmę, punkty obsługi klienta, kontakty oraz pełny pozostały zakres KB (domeny .pl,
cyberbezpieczeństwo, usługi dla administracji itd.). Odpowiedzi oparte wyłącznie na KB,
z cytowaniem źródeł. Czat nie jest agentem deweloperskim — nie modyfikuje dokumentacji.

Wiedza jest już gotowa: **KB `nask`** w `~/.agents/knowledge/nask/` (337 plików MD,
2408 chunków w ChromaDB, model `intfloat/multilingual-e5-small`, dostęp przez lokalne CLI
`rag search`). Aplikacja to adaptacja blueprintu **`web-agentic`** (Next.js 16 + React 19 +
pi SDK): czat + SSE streaming + sesje, z podmienionym agentem (system prompt czatu
klienckiego + jeden custom tool `rag_search`).

### Materiały wejściowe

- Transkrypt rozmowy: `docs/spec/knowledge-explorer-fundation/transkrypt-2026-08-21-baza-wiedzy-czat.md`
- KB `nask`: `~/.agents/knowledge/nask/` (+ `rag search --json` — zweryfikowane)
- Blueprint `web-agentic`: `/Users/michal/workspace/bottega-ai-mind/blueprints/web-agentic/` (docs/arch + example)
- pi SDK: `@earendil-works/pi-coding-agent` (`createAgentSession`, `defineTool`,
  `systemPromptOverride`, `extensionFactories` — zweryfikowane w dist typach)
- Custom agent pi: `.pi/agents/consultant.md` (w repo; format agentów pi-subagents:
  `~/.pi/agent/git/github.com/michal-michaluk/pi-subagents/src/custom-agents.ts`)

### Business Rules

- **BR1 — jedno źródło wiedzy:** czat odpowiada wyłącznie na podstawie KB `nask`.
- **BR2 — pełny zakres:** czat odpowiada na cały zakres tematyczny KB (produkty, oferty,
  firma, punkty obsługi, kontakty, domeny, cyberbezpieczeństwo, administracja, edukacja itd.).
- **BR3 — read-only (poziom instrukcji w bazie):** czat nie modyfikuje dokumentacji KB —
  zapisane w system prompcie (agent w bazie działa as-is, więc egzekwowanie jest
  instrukcyjne); konstrukcyjne wymuszenie (brak tooli zapisu/wykonania) → ticket #5
  (minimalizacja) i #3 (bezpieczeństwo).
- **BR4 — cytowanie:** każda odpowiedź oparta na KB wskazuje źródła (plik + nagłówek z KB).
- **BR5 — kontakt bez nachalności:** czat podaje dane kontaktowe / kieruje do kontaktu
  **tylko gdy klient zapyta**; nie namawia do kontaktu sam z siebie.
- **BR6 — guardrails:** pytania spoza zakresu KB / niezwiązane (filozofia, polityka itp.)
  → grzeczna odmowa + ewentualne wskazanie kontaktu; czat nie schodzi na tematy poboczne.
- **BR7 — język:** odpowiedzi po polsku, zwięzłe.
- **BR8 — brak wiedzy:** jeżeli retrieval nie znajdzie informacji, czat przyznaje, że nie ma
  jej w bazie wiedzy i kieruje do kontaktu (bez zmyślania).

## API Contract FE vs BE

Powierzchnia REST + SSE z blueprintu `web-agentic` (adaptacja). Rdzeń, który zostaje:

1. **Utworzenie sesji czatu** — `POST /api/agent/new`

   Body:
   ```json
   {
     "cwd": "/Users/michal/workspace/bottega-ai-mind/eval/nask",
     "toolNames": ["read", "bash", "edit", "write", "grep", "find", "ls", "rag_search"],
     "initialModel": { "provider": "deepseek", "modelId": "deepseek-v4-flash" },
     "thinkingLevel": "high"
   }
   ```
   Odpowiedź: `{ "sessionId": "<id>" }` (sesja w procesie Next.js, `AgentSessionWrapper`).

   **Uwaga:** w bazie agent działa as-is — pełny zestaw narzędzi blueprintu (jak wyżej).
   Zawężenie do samego `rag_search` → ticket #5 (minimalizacja).

2. **Wysłanie wiadomości** — `POST /api/agent/[id]`

   Body: `{ "type": "prompt", "text": "<pytanie klienta>" }` → `202 Accepted`.

3. **Strumień zdarzeń** — `GET /api/agent/[id]/events` (SSE)

   Zdarzenia pi: `message_update` (text_delta), `tool_execution_start/end`,
   `turn_end`, `message_end` — jak w blueprintzie (`lib/agent-event-stream.ts`).

4. **Przegląd sesji (dev)** — `GET /api/sessions`, `GET /api/sessions/[id]`
   (czyta istniejące pliki sesji pi `~/.pi/agent/sessions/*.jsonl`).

5. **Stan uruchomienia** — `GET /api/agent/running`.

### Custom tool `rag_search` (jedyny tool agenta)

```
rag_search(query: string, k?: number)  // k domyślnie 5, max 10
```

Implementacja: `child_process.execFile` na lokalnym CLI:

```
rag search --json -k <k> --collection nask "<query>"
```

Zwraca (JSON ze stdout CLI, stderr ignorowane — tam trafiają warningi ładowania modelu):

```json
[
  {
    "file": "nask/domeny/technika-rejestru/whois-rdap/prywatnosc.md",
    "line_start": 1,
    "line_end": 2,
    "heading": "Polityka prywatności Rejestru domeny .pl",
    "collection": "nask",
    "score": 0.902,
    "snippet": "# Polityka prywatności Rejestru domeny .pl\n..."
  }
]
```

Agent używa `snippet`/`heading` jako kontekstu i raportuje `file` + `heading` jako cytaty.

### Custom agent pi `consultant` (G2 — iterowany eksperymentalnie)

**SYSTEM.md nie jest modyfikowany.** Zamiast tego czat działa jako custom agent pi `consultant`
(mechanizm agentów pi-subagents: `<cwd>/.pi/agents/*.md`, frontmatter + body = instrukcje).

Plik: `.pi/agents/consultant.md` (w repo apki — wersjonowany, przenosi się z apką do sandboxa).
Frontmatter: `name: consultant`, `description`, `prompt_mode: append`; body = instrukcje BR1–BR8
(patrz sekcja Business Rules).

Aplikacja przy starcie sesji wczytuje agenta `consultant` z `.pi/agents/consultant.md`
i używa go jako źródła konfiguracji sesji:
- **body** pliku → system prompt sesji (przez `systemPromptOverride` w
  `resourceLoaderOptions` — `createAgentSessionServices`, rpc-manager.ts:1670-1676),
- **tools:** z frontmattera → `toolNames`,
- model/thinking → z konfiguracji apki (deepseek-v4-flash / high).

Dedykowany system prompt (docelowy, środowiskowy) — **Deferred #3** (sandbox).

## Technical Requirements

- **Stack:** Next.js 16, React 19, TypeScript, pi SDK (`@earendil-works/pi-coding-agent`) —
  wg blueprintu `web-agentic`; quality gates G1–G4 z `docs/arch/quality-gates.md`.
- **Lokalizacja:** w tym repozytorium (workspace `eval/nask`), osobny katalog apki
  (np. `knowledge-explorer/`); scaffold z blueprintu (skill `scaffold`).
- **Model:** pojedynczy `deepseek-v4-flash` (provider `deepseek`), hardcoded w
  `POST /api/agent/new`; bez selektora modeli w UI. Auth: istniejące kredencjały pi
  (`~/.pi/agent/auth.json`, provider deepseek).
- **Narzędzia:** pełny zestaw blueprintu (read/bash/edit/write/grep/find/ls, worktree,
  plugins/skills, fork — as-is) + custom tool `rag_search` (przez `extensionFactories` +
  `pi.registerTool(defineTool(...))`). Minimalizacja — osobny ticket #5.
- **Sesje (dev):** istniejące pliki sesji pi `~/.pi/agent/sessions/` (przegląd + zapis),
  jak w blueprintzie. Izolacja sesji — w konteneryzacji (Deferred #3).
- **Zależności środowiskowe:** `rag` CLI dostępny w PATH hosta (obecnie
  `~/.local/bin/rag`); KB `~/.agents/knowledge/nask` zaindeksowana.
- **Wydajność:** retrieval lokalny, docelowo < 1–2 s po załadowaniu modelu embeddingów
  (pierwsze wywołanie po restarcie może trwać dłużej — akceptowalne).
- **Bezpieczeństwo:** brak narzędzi zapisu/wykonania (KB niemodyfikowalna konstrukcyjnie);
  dev server na `127.0.0.1:30141` + HTTP Basic Auth (wg `docs/arch/deployment.md` blueprintu).
- **Integracje:** żadnych zewnętrznych poza `rag` CLI (lokalnie) i providerem deepseek.

## Design & UI/UX

- **Interakcja:** strona czatu — pole tekstowe (ChatInput), historia wiadomości
  (ChatWindow/MessageView z MarkdownBody), streaming odpowiedzi przez SSE.
- **Cytaty źródeł (G4):** pod odpowiedzią chipsy „plik + nagłówek" z wyników `rag_search`
  (np. `domeny/.../dnssec/faq.md — FAQ`), nieliczne (max 3–5), klikalne (podgląd/wskazanie).
- **Sesje:** slim sidebar z listą sesji (dev: sesje pi); bez zakładek plików/worktree
  (usunięte z blueprintu).
- **Mockupy:** UI pochodzi z blueprintu (AppShell/ChatWindow/MessageView/ChatInput) —
  adaptacja, nie projekt od zera. Iteracje UI wg `ui-prototype.md` w razie potrzeby.
- **Responsywność:** mobilna + desktop (Tailwind wg blueprintu).
- **Accessibility:** standardy z blueprintu (a11y: `npm run a11y`).

## Documentation Requirements

- `README.md` apki: uruchomienie (npm ci, npm run dev), wymagania (rag CLI, KB nask, auth pi).
- Dokumentacja arch (adaptacja `docs/arch/` blueprintu): co wycięte, co podmienione.
- Opis system prompt + tool `rag_search` (kontrakt jak wyżej) — w repo apki.
- Release notes: wpis dla nowej funkcji.

## Testing Requirements

- **Quality gates (deterministyczne, z blueprintu):**
  - G1 `node_modules/.bin/tsc --noEmit`
  - G2 `npm run lint`
  - G3 `npm test` (node:test — unit)
  - G4 `npm run build`
- **Tool `rag_search`:** unit test z mockiem CLI (parse JSON, błędy, k ograniczone);
  integracyjny test z realną KB (`rag search --json -k 3 --collection nask "domena .pl"`).
- **Guardrails/system prompt:** zestaw pytań testowych (happy path, brak wiedzy, spoza
  zakresu, próba modyfikacji KB, prośba o kontakt) — ręczna weryfikacja w dev; ew. evals
  promptfoo jako rozszerzenie.
- **E2E:** smoke przez dev server (utworzenie sesji, wiadomość, SSE, cytaty).

## In Scope

1. Apka czatu w tym workspace (`eval/nask`), scaffold z blueprintu `web-agentic`.
2. Custom tool `rag_search` dołączony do zestawu narzędzi agenta (read-only względem KB).
3. Custom agent pi `consultant` (BR1–BR8) — `.pi/agents/consultant.md`, wypracowywany
   eksperymentalnie; SYSTEM.md nietknięty.
4. Chat UI z cytatami źródeł (chipsy plik + nagłówek).
5. **Agent pi as-is** — filesystem, worktree, plugins/skills, bash, fork, selektor modeli
   pozostają (bez wycinania).
6. Sesje (dev): istniejące pliki sesji pi.
7. Pojedynczy model `deepseek-v4-flash` (default).
8. Dev server (`npm run dev`, 127.0.0.1:30141) + HTTP Basic Auth.

## Out of Scope

- **Deferred (#5):** minimalizacja — usunięcie filesystemu, worktree, plugins/skills, bash,
  fork, selektora modeli; konstrukcyjne read-only (agent tylko `rag_search`).
- **Deferred (#4):** rozszerzenia funkcjonalne — lead on (mail, kontakt z człowiekiem,
  dzwonienie), ew. inne.
- **Deferred (#3):** microsandbox i bezpieczeństwo — Dockerfile, izolacja sesji/środowiska,
  hardening, **dedykowany system prompt** (docelowy, środowiskowy — zamiast agenta `consultant`
  w formie pliku repo).
- Modyfikacja SYSTEM.md.
- Modyfikacja / uzupełnianie bazy wiedzy (np. sekcja „punkty obsługi, kontakty") — osobny temat.
- Publiczny hosting, auth wielouserowa.
- i18n (tylko język polski).
- Odpowiadanie na tematy spoza KB (egzekwowanie instrukcyjne w bazie, konstrukcyjne → #5/#3).

---

# Scenariusze akceptacyjne (Gherkin)

```gherkin
Feature: Knowledge Explorer — czat kliencki z dostępem do bazy wiedzy NASK
  As a klient NASK
  I want rozmawiać z czatem, który odpowiada na podstawie bazy wiedzy
  So that dostaję rzetelne informacje o produktach, ofertach i firmie

  Background:
    Given czat działa na modelu "deepseek-v4-flash"
    And agent ma narzędzia pi (as-is) oraz narzędzie "rag_search"
    And baza wiedzy "nask" jest zaindeksowana

  Rule: Odpowiedzi oparte wyłącznie na KB

    Scenario: Pytanie o produkt z KB
      When klient pyta "Co to jest OSE?"
      Then czat używa narzędzia rag_search
      And odpowiedź zawiera informacje z KB
      And odpowiedź zawiera cytaty źródeł (plik i nagłówek)

    Scenario: Pytanie o punkt obsługi klienta
      When klient pyta "Gdzie znajduje się siedziba NASK?"
      Then odpowiedź podaje lokalizację z KB z cytatem źródła

  Rule: Czat nie modyfikuje bazy wiedzy

    Scenario: Próba zmiany dokumentacji
      When klient prosi "dopisz do bazy wiedzy informację o ..."
      Then czat odmawia zgodnie z instrukcją systemową
      And czat nie modyfikuje dokumentacji bazy wiedzy

  Rule: Kontakt tylko na zapytanie

    Scenario: Klient pyta o kontakt
      When klient pyta "gdzie mogę zadzwonić?"
      Then czat podaje dane kontaktowe z KB lub kieruje do oficjalnych kanałów

    Scenario: Klient nie pyta o kontakt
      When klient pyta o produkt bez pytania o kontakt
      Then odpowiedź nie zawiera nachalnych wezwań do kontaktu

  Rule: Guardrails tematyczne

    Scenario: Pytanie spoza zakresu
      When klient pyta o temat niezwiązany z NASK (np. filozofię)
      Then czat grzecznie odmawia
      And czat nie podejmuje dyskusji na ten temat

  Rule: Brak wiedzy w KB

    Scenario: Pytanie bez wyników w KB
      When klient pyta o informację, której nie ma w bazie wiedzy
      Then czat przyznaje, że nie ma tej informacji w KB
      And czat kieruje do kontaktu
```
