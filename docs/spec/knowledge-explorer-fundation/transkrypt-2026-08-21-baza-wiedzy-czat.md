# Transkrypt rozmowy — baza wiedzy i czat dla klientów

- **Data:** 2026-08-21
- **Temat:** Pierwsze kroki — baza wiedzy o produktach/ofertach/firmie + webowy czat z dostępem do tej bazy
- **Status:** notatki wstępne, przed specyfikacją

---

## Przebieg rozmowy

**A:** To chcielibyśmy mieć w pierwszej kolejności jakąś bazę wiedzy na temat naszych produktów, ofert, czy firmy. Bo klient może pytać tam o różne rzeczy — gdzie jest najbliższy [punkt], kontakty i tak dalej.

**B:** Dokładnie. Kontakty, gdzie zadzwonić, albo gdzie jest — nie wiem — jeżeli jest punkt jakiejś obsługi klienta, to gdzie te punkty są.

**A:** Pierwsze to baza wiedzy.

**B:** Dobra, to jest na pewno fajny punkt do zrobienia.

**A:** Druga rzecz, która tu na pewno będzie potrzebna: chcielibyśmy mieć frontend webowy — pewnie stronkę — na której klient może pogadać sobie z czatem, który będzie miał dostęp do tej bazy wiedzy, po prostu wewnętrznie.

**B:** Tak, najlepiej, żeby to było ograniczone.

**A:** Tak, tak. Czyli żeby nie był to deweloperski agent, tylko stricte do gadania i lead on, żeby też nie zmieniał tej dokumentacji, którą mamy w bazie wiedzy.

**B:** To na pewno się zgadza.

**A:** Ja myślę, że nie musimy teraz tego specyfikować, ale docelowo — wiadomo — mógłby tam nawet, jak użytkownik poprosi, wysłać maila, czy skontaktować się z kimś, czy zadzwonić do człowieka. Ale to na razie olejmy — to będzie czysta eksploracja, podstawa dopiero.

**B:** Tak, tak, na pewno.

**A:** No, dobrze by było, żeby to było w miarę zwarte i zintegrowane, żeby nam nie płynął ten czat, nie dawało się wciągać w dyskusje filozoficzne, omawianie Hitlera i tak dalej. Czyli żeby ta instrukcja się trzymała kupy i była sprecyzowana.

**B:** Dobra. To jeszcze sobie też eksperymentalnie — bo tę instrukcję trzeba wypracować na pewno doświadczalnie.

**A:** No, dobra, dobra. I na razie to zrobimy jako osobną apkę — nie będziemy tego wbudowywać w żadną istniejącą. Na razie zróbmy to totalnie jako osobne, osobną apkę. I na tym się skupmy.

---

## Ustalenia (skrót)

1. **Baza wiedzy** — produkty, oferty, firma; lokalizacje punktów obsługi klienta, kontakty (gdzie zadzwonić). To pierwszy punkt do zrobienia.
2. **Webowy czat** — frontend, na którym klient rozmawia z czatem mającym (wewnętrzny) dostęp do bazy wiedzy.
3. **Ograniczenia czatu** — nie deweloperski agent; stricte rozmowa + lead on; nie modyfikuje dokumentacji w bazie wiedzy; zwarta, sprecyzowana instrukcja, żeby nie schodził na tematy filozoficzne itp.
4. **Poza zakresem (na razie)** — wysyłanie maila, kontakt z człowiekiem, dzwonienie. Docelowo możliwe, teraz czysta eksploracja.
5. **Wypracowanie instrukcji** — eksperymentalnie, na podstawie doświadczenia.
6. **Architektura** — osobna apka, nic nie wbudowujemy w istniejące systemy.

---

## Uzupełnienie — istniejąca baza wiedzy `nask`

Baza wiedzy o NASK **już istnieje** i ma posłużyć jako wiedza dla czatu (punkt 1 z rozmowy jest więc zrealizowany — nie budujemy nowej KB).

- **Lokalizacja:** `~/.agents/knowledge/nask/` (knowledge home: `~/.agents/knowledge/`, stan na 2026-08-20)
- **Zawartość:** 337 plików MD — instytucja i ustrój, historia, organizacja i ludzie, rejestr domen .pl, cyberbezpieczeństwo publiczne (KSC/NIS2/CERT Polska) i komercyjne (produkty PIB, oferta NASK S.A., OSiC), usługi dla administracji (EZD RP, OSE, tożsamość cyfrowa), edukacja i społeczeństwo, infrastruktura sieciowa, badania i rozwój, klienci i segmenty, glossar + index
- **Indeks:** 2408 chunków w ChromaDB (`.rag/chromadb`, ~22 MB), model `intfloat/multilingual-e5-small`, ostatni indeks 2026-08-20
- **Dostęp:** lokalne wyszukiwanie semantyczne przez CLI `rag search` (offline, cross-lingual) — zweryfikowane, zwraca trafne chunki (np. adres/dane rejestrowe, oddział Białystok, telco)
- **Konsekwencja dla spec:** czat „knowledge explorer” będzie odpytował istniejącą KB przez `rag search` (backend retrieval), a nie budował własną. Ewentualne uzupełnienia KB (np. punkty obsługi klienta, kontakty) to osobny temat.
