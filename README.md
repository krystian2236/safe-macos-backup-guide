# Bezpieczny backup macOS i GitHub

Publiczny poradnik tworzenia, sprawdzania i odtwarzania kopii. Przykładowe polecenia zakładają powłokę Zsh w macOS. Najpierw ustal zakres danych, potem wykonaj diagnostykę tylko do odczytu. Zapis i usuwanie są osobnymi, świadomymi krokami.

<a id="index"></a>
## Przejdź do

| Etap | Rozdział |
| --- | --- |
| Przygotowanie | [Jak używać](#how-to-use) · [Bezpieczeństwo](#security) · [Struktura i kategorie](#structure) |
| Wykonanie | [Procedura backupu](#backup) · [Weryfikacja](#verification) · [Projekty](#projects) |
| Źródła | [GitHub](#github) · [SSH](#ssh) · [Konfiguracje aplikacji i Safari](#safari) |
| Dalsza praca | [Odtwarzanie](#restore) · [Problemy i rozwiązania](#troubleshooting) · [Publikowanie](#publishing) · [Przykład](#example) |

<a id="how-to-use"></a>
## Jak używać tej instrukcji

Zapis `<...>` oznacza placeholder: własną wartość, którą trzeba podstawić przed wykonaniem polecenia. Znaków `<` i `>` **nie wpisuje się** do finalnych poleceń; powłoka mogłaby potraktować je jak przekierowanie. Przykład: `<BACKUP_VOLUME>` zastąp rzeczywistą ścieżką zamontowanego dysku. Ścieżki zawierające spacje ujmuj w cudzysłowy. Kod w tym poradniku jest wzorem, nie gotowym skryptem do bezrefleksyjnego uruchomienia.

Wszystkie poniższe dane są **fikcyjne**:

```text
<HOME> = /Users/alex
<BACKUP_VOLUME> = /Volumes/Backup
<GITHUB_USERNAME> = alex-dev
<REPOSITORY_NAME> = ExampleApp
```

Pozostałe placeholdery nazywają kategorię, projekt, plik, repozytorium lub wybrany snapshot. Przed kopiowaniem sprawdź, czy podstawiona ścieżka źródła i celu wskazuje właściwe miejsce.

`<BRANCH_NAME>` oznacza nazwę gałęzi, `<COMMIT_SHA>` identyfikator commita, `<PROFILE_ID>` identyfikator profilu, a `<BUNDLE_IDENTIFIER>` identyfikator aplikacji. Traktuj je jako przykładowe etykiety, nigdy jako wartości z cudzego środowiska.

[↑ Powrót do spisu](#index)

<a id="security"></a>
## Bezpieczeństwo

- **Nigdy nie publikuj** prywatnego klucza SSH, tokenów, haseł ani plików credentials. Plik `*.pub` jest publiczną częścią pary kluczy, ale także jego publikację oceniaj świadomie.
- Kopię zawierającą sekrety przechowuj na zaszyfrowanym nośniku; ogranicz dostęp do nośnika i kopii. Publiczne repozytorium GitHub nie jest miejscem na taki backup.
- Przed publikacją README lub poradnika skontroluj tekst i historię commitów pod kątem sekretów oraz danych identyfikujących.
- Zaczynaj od diagnostyki tylko do odczytu. Zapis wykonuj po sprawdzeniu źródła, celu, filtrów i wolnego miejsca. **DELETE zawsze na końcu**, po pełnej weryfikacji i upewnieniu się, że nie usuwasz jedynej kopii.
- Nie używaj `curl | sh`. `sudo` stosuj tylko wtedy, gdy jest naprawdę potrzebne, z jednoznaczną ścieżką i zakresem operacji.

Przy błędzie zatrzymaj usuwanie i zachowaj działające źródło. Zobacz [procedurę](#backup) i [diagnostykę](#troubleshooting).

[↑ Powrót do spisu](#index)

<a id="structure"></a>
## Struktura backupu i kategorie

```text
<BACKUP_VOLUME>/
├── CURRENT/
│   ├── Codex/
│   ├── Ollama/
│   ├── AI-Tools/
│   ├── Zsh/
│   ├── Homebrew/
│   ├── Projects/
│   ├── iTerm2/
│   ├── SSH/
│   └── Safari/
└── ARCHIVE/
    └── <CATEGORY>/<YYYY-MM-DD>/
```

`CURRENT` to najnowsza **zweryfikowana** kopia potrzebna do szybkiego odtworzenia. `ARCHIVE` przechowuje starsze, zweryfikowane punkty historyczne. Wybieraj datę stanu archiwizowanego; nie łącz różnych snapshotów w jednym katalogu. Przed zastąpieniem `CURRENT/<CATEGORY>/` zabezpiecz poprzedni ważny stan w `ARCHIVE` i sprawdź go. Nie nadpisuj jedynej dobrej kopii.

| Kategoria | Co zachować i dlaczego | Czego zwykle nie kopiować i dlaczego | Jak zweryfikować i odtworzyć |
| --- | --- | --- | --- |
| Codex | Używaną konfigurację, własne instrukcje i skrypty; odtwarzają sposób pracy. | Cache, sesje i pliki uwierzytelnienia bez osobnej decyzji; bywają odtwarzalne lub wrażliwe. | SHA256, porównanie plików i uruchomienie narzędzia; przywróć tylko potrzebne ustawienia po porównaniu z aktywnymi. |
| Ollama | Modelfile, własne szablony i ustawienia; opisują konfigurację modeli. | Pobrane wagi, jeśli można je ponownie pobrać; zajmują dużo miejsca. | SHA256 i kontrola Modelfile; przywróć pliki, a wagi pobierz lub odtwórz z osobnej kopii. Sam Modelfile nie zawiera wag. |
| AI-Tools | Własne skrypty, definicje i dokumentację; mogą być unikalne. | Środowiska wirtualne, cache, logi i sekrety; zwykle są odtwarzalne lub wrażliwe. | SHA256, kontrola składni i mały test działania; przywróć wybrane skrypty oraz zależności. |
| Zsh | Używane pliki konfiguracji, funkcje i własne skrypty; odtwarzają powłokę. | Historię poleceń i cache bez osobnej potrzeby; mogą ujawniać dane. | SHA256 i `zsh -n` na przywracanym pliku; po porównaniu odtwórz tylko potrzebne wpisy. |
| Homebrew | Listy pakietów, casków, tapów lub Brewfile; pozwalają odtworzyć instalacje. | Cache pobrań i butelki, jeśli są dostępne; zajmują miejsce. | SHA256 i odczyt list; po przywróceniu zainstaluj wybrane pozycje i sprawdź wersje. |
| Projects | Kod, `.git`, pliki projektu i unikalne artefakty; mogą nie istnieć gdzie indziej. | Wyłącznie potwierdzone buildy i cache; szczegóły w [Projektach](#projects). | SHA256, checksum dry-run i testy Git; przywróć projekt w osobnym miejscu, potem sprawdź historię i build. |
| iTerm2 | Preferences, profile, snippets, shell integration i własne skrypty; odtwarzają układ pracy. | Cache, historię i środowiska narzędzi; bywają duże lub wrażliwe. | SHA256, `plutil -lint` dla plist i kontrola widoczności profili; przywróć tylko potrzebne pliki. |
| SSH | Potrzebne klucze i konfigurację; mogą być niemożliwe do ponownego wygenerowania z tą samą tożsamością. | Sockety agenta i pliki runtime; nie są trwałą konfiguracją. | SHA256, uprawnienia i fingerprint **własnego** klucza publicznego; przywróć z restrykcyjnymi uprawnieniami. |
| Safari | Potrzebne preferences, profile, konfiguracje rozszerzeń lub snippets; odtwarzają ustawienia. | Cache, historię, runtime i całe kontenery bez audytu; mogą być ogromne lub wrażliwe. | SHA256, `plutil -lint` i kontrola ustawień w aplikacji; odtwarzaj konkretny wpis lub plik. |

Lista kategorii jest szablonem. Przed włączeniem danych do kopii oceń ich poufność, rozmiar i możliwość ponownego pobrania.

[↑ Powrót do spisu](#index)

<a id="backup"></a>
## Procedura backupu

Przykładowe `<SOURCE>` i `<DEST>` oznaczają odpowiednio katalog źródła oraz staging. Podstaw je dopiero po diagnostyce. Polecenia z `--delete` w tej sekcji mają `-n`, więc niczego nie usuwają; obejrzyj listę zmian. Właściwe usuwanie nie jest częścią kopiowania.

1. **Sprawdź montowanie dysku.** `mount` i `df -h <BACKUP_VOLUME>` muszą pokazać oczekiwany wolumin, nie pusty katalog lokalny.
2. **Sprawdź źródło.** `ls -ld <SOURCE>` oraz lista potrzebnych plików potwierdzają jego istnienie i zakres. Dla repozytoriów sprawdź [Git](#github).
3. **Sprawdź wolne miejsce.** Porównaj `du -sh <SOURCE>` z `df -h <BACKUP_VOLUME>`, uwzględniając dotychczasowe snapshoty i zapas.
4. **Utwórz staging** `<CATEGORY>.incoming` na tym samym woluminie, w miejscu docelowym. Upewnij się, że nazwa nie wskazuje ważnej starszej kopii. Staging oddziela kopię nieukończoną od `CURRENT` lub `ARCHIVE`.
5. **Skopiuj dane.** Po audycie filtrów użyj na przykład `rsync -a <SOURCE>/ <DEST>/`. Na macOS `-a` zachowuje strukturę, uprawnienia i czasy w zakresie obsługiwanym przez `rsync`; potrzebę ACL/xattr oceniaj osobno.
6. **Napraw problemy z plikami specjalnymi.** Jeśli kopiowanie zgłasza socket lub inny plik runtime, zidentyfikuj go i wyklucz tylko ten element; uruchom `rsync` ponownie. Zobacz [problem socketów](#troubleshooting).
7. **Wykonaj checksum dry-run.** `rsync -acn --delete --itemize-changes <SOURCE>/ <DEST>/` powinien dać zero różnic treści i listy plików po uwzględnieniu tych samych filtrów po obu stronach. Nie przechodź dalej przy niewyjaśnionej różnicy.
8. **Utwórz `SHA256SUMS`** w katalogu staging według [wzoru](#verification).
9. **Zweryfikuj SHA256.** Każdy wpis musi dać `OK`, a liczba wpisów ma odpowiadać liczbie plików danych.
10. **Wykonaj dodatkowe testy Git**, jeśli kopia zawiera repozytorium: sprawdź branche, worktree, niepushowane commity i `git fsck --full` dla krytycznych repozytoriów. Zobacz [GitHub](#github).
11. **Dopiero wtedy zmień nazwę `.incoming` na finalną.** Przed `mv` potwierdź, że ścieżka finalna nie istnieje albo że poprzedni stan został osobno, poprawnie zarchiwizowany. Zmiana nazwy w obrębie woluminu szybko ujawnia gotową kopię.
12. **Stare źródło wolno usunąć dopiero po pełnej weryfikacji** `CURRENT` lub `ARCHIVE`, zgodności hashy, ocenie unikalnych danych i świadomej decyzji. Usunięcie jest ostatnią, osobną operacją.

Jeśli `rsync` przerwie kopiowanie, zwykle zachowaj `.incoming`, popraw konkretny problem i uruchom kopiowanie ponownie. Po finalizacji ponownie sprawdź manifest z finalnego katalogu. Dalsze warunki opisuje [problem finalizacji](#troubleshooting).

[↑ Powrót do spisu](#index)

<a id="verification"></a>
## Weryfikacja backupu

Z katalogu snapshotu lub `.incoming` wygeneruj manifest ze **względnymi** ścieżkami:

```zsh
find . -type f ! -name SHA256SUMS -print0 \
  | sort -z \
  | xargs -0 shasum -a 256 >| SHA256SUMS
shasum -a 256 -c SHA256SUMS
```

Manifest nie zawiera samego siebie. Przy 100 plikach danych ma 100 wpisów, a cały katalog zawiera wtedy 101 zwykłych plików. Gdy wyłączasz inne pliki z manifestu, odnotuj to jawnie i odpowiednio skoryguj liczenie. Symboliczne linki, sockety i metadane nie są zwykłymi plikami objętymi powyższym `find`; sprawdź je osobno. Przy nazwach plików zawierających znak nowej linii zastosuj narzędzie obsługujące taki przypadek i przetestuj odczyt manifestu.

Porównanie treści i listy plików przed utworzeniem manifestu:

```zsh
rsync -acn --delete --itemize-changes <SOURCE>/ <DEST>/
```

Zero output oznacza brak różnic treści i dodatkowych/brakujących plików w porównywanym zakresie. Po utworzeniu manifestu wyklucz go z porównania, na przykład `--exclude=/SHA256SUMS`, stosując identyczne inne filtry. Wynik dry-run nie potwierdza uprawnień, ACL, xattr ani poprawności wyboru źródła; sprawdź je osobno. `shasum -c` uruchamiaj z katalogu, do którego odnoszą się ścieżki manifestu.

[↑ Powrót do spisu](#index)

<a id="projects"></a>
## Projekty

Zachowuj kod źródłowy, `.git`, lokalne branche, niepushowane commity, metadane worktree, pliki projektu oraz unikalne artefakty, których nie da się odtworzyć. Pełne `.git` często jest jedyną kopią lokalnej historii. Zobacz [GitHub](#github).

Można rozważyć pomijanie `build/`, `builds/`, `DerivedData/`, `node_modules/`, `.gradle/`, `.godot/`, `cache/`, `dist/` i `export_templates/`. **Nigdy nie wykluczaj ich automatycznie bez audytu.** Gotowy APK, IPA lub inny artefakt może być jedyną zachowaną wersją.

Najpierw zbadaj rozmiary i kandydatów na dane generowalne:

```zsh
du -sh <PROJECT>/*
find <PROJECT> -type d \( -name build -o -name builds -o -name DerivedData -o -name node_modules -o -name .gradle -o -name .godot -o -name cache -o -name dist -o -name export_templates \) -print
```

Pierwsze polecenie pomija ukryte pozycje, dlatego obejrzyj je także osobno. `find` tylko wskazuje kandydatów; każdą ścieżkę przejrzyj przed dodaniem filtra. Po kopiowaniu wykonaj [porównanie checksum](#verification) z tymi samymi filtrami.

[↑ Powrót do spisu](#index)

<a id="github"></a>
## GitHub a backup

Lokalne repozytorium zawiera katalog roboczy i `.git`. GitHub jako `remote` przechowuje tylko dane wypchnięte do usługi. Niezależny backup GitHub to osobna kopia danych hostowanych, w tym potrzebnych metadanych pobranych przez API lub eksport. Żaden z tych trzech zakresów nie zastępuje automatycznie pozostałych.

`git clone` może nie odtworzyć lokalnych branchy i commitów niewypchniętych na serwer, worktree, dangling objects, issues, pull requestów, ustawień repozytorium ani branch protections. Dlatego pełne `.git` bywa ważne. Jeśli korzystasz z worktree, sprawdź powiązania między katalogami i zachowaj komplet metadanych; pojedynczy katalog roboczy może nie wystarczyć.

Diagnostyka lokalnego repozytorium jest tylko do odczytu:

```zsh
git -C <PROJECT> status --short --branch
git -C <PROJECT> branch -a -vv
git -C <PROJECT> worktree list
git -C <PROJECT> log --branches --not --remotes --oneline
git -C <PROJECT> fsck --full
```

Zapisz wyniki kontroli bez publikowania prywatnych adresów remote. `dangling` nie oznacza automatycznie uszkodzenia. Nie uruchamiaj `git gc`, `prune` ani nie kasuj packów przed potwierdzeniem, że potrzebne dane zostały odtworzone gdzie indziej.

[↑ Powrót do spisu](#index)

<a id="ssh"></a>
## SSH

**NIGDY nie publikuj zawartości prywatnego klucza SSH.** `<SSH_PRIVATE_KEY>` oznacza prywatny plik, a `<SSH_PUBLIC_KEY>` odpowiadający mu plik `*.pub`. Klucz publiczny można przekazać usłudze, ale prywatny pozwala uwierzytelniać się jako właściciel i musi pozostać tajny.

Kopiuj potrzebne klucze i konfigurację SSH wyłącznie na zaszyfrowany nośnik. Przed kopiowaniem sprawdź, czy pliki mają właściwego właściciela i uprawnienia. Po odtworzeniu katalog SSH powinien być dostępny tylko właścicielowi, prywatny klucz zwykle mieć tryb `600`, a publiczny `644`; skonfiguruj prawa tylko dla konkretnych własnych plików.

```zsh
ls -ld <HOME>/.ssh
ls -l <SSH_PRIVATE_KEY> <SSH_PUBLIC_KEY>
ssh-keygen -lf <SSH_PUBLIC_KEY>
```

Ostatnie polecenie oblicza fingerprint **własnego klucza publicznego**. Porównaj go lokalnie z wcześniej zaufanym zapisem lub ustawieniem konta; nie umieszczaj rzeczywistego fingerprintu w publicznym poradniku. Po odtworzeniu sprawdź SHA256 plików, uprawnienia i połączenie z właściwą usługą.

[↑ Powrót do spisu](#index)

<a id="safari"></a>
## Konfiguracje aplikacji i Safari

Nie kopiuj całych wielogigabajtowych kontenerów bez audytu. Często wystarczą preferences, profile, konfiguracje rozszerzeń i snippets. Cache, historia i pliki runtime mogą być zbędne, duże albo wrażliwe. Ustal lokalizację ustawień danej wersji aplikacji i przetestuj odtworzenie na małym zakresie.

Gdy modyfikujesz plist, zmieniaj **tylko konkretny wpis**. Najpierw zachowaj zweryfikowaną kopię, potem zapisz zmianę przez plik tymczasowy z zachowaniem uprawnień. Na końcu zawsze wykonaj `plutil -lint` na zmienionym pliku. W przypadku Safari i rozszerzeń sprawdź też wynik `pluginkit -m -A -D`; wpis może wskazywać na osadzone rozszerzenie aplikacji, a nie osobną aplikację. Zobacz [problemy](#troubleshooting).

[↑ Powrót do spisu](#index)

<a id="restore"></a>
## Procedura odtwarzania

1. Wybierz `CURRENT/<CATEGORY>/` albo konkretny `ARCHIVE/<CATEGORY>/<YYYY-MM-DD>/`. Potwierdź montowanie dysku i właściwą datę snapshotu.
2. Z katalogu snapshotu uruchom `shasum -a 256 -c SHA256SUMS`; każda pozycja musi dać `OK`. Sprawdź także kompletność potrzebnych plików i metadanych.
3. Porównaj snapshot z bieżącą konfiguracją: wersje plików, uprawnienia, zakres danych i lokalną historię Git. Zachowaj aktualny stan do ewentualnego cofnięcia.
4. Nie nadpisuj działającego środowiska bez porównania. Wybierz najmniejszy wymagany zakres; najpierw odtwórz go w osobnym miejscu, jeśli to możliwe.
5. Odtwórz tylko potrzebny plik, katalog lub repozytorium. Sekrety przenoś wyłącznie w bezpiecznym środowisku i przywróć odpowiednie uprawnienia.
6. Po odtworzeniu uruchom właściwą aplikację lub usługę i sprawdź działanie: dla Zsh składnię, dla Git branche i historię, dla plist `plutil -lint`, dla SSH uprawnienia i połączenie. Odnotuj wynik i ewentualne braki.

Gdy SHA256 nie przechodzi, nie uznawaj snapshotu za sprawdzony. Zachowaj go do diagnozy i wybierz inny zweryfikowany punkt.

[↑ Powrót do spisu](#index)

<a id="troubleshooting"></a>
## Problemy i rozwiązania

Zacznij od [procedury backupu](#backup), [weryfikacji](#verification) i [odtwarzania](#restore). Poniższa diagnostyka nie usuwa danych; polecenia zapisujące są opisane warunkami.

### 1. `rsync -E`: `Permission denied` lub `._*`

**Objaw:** Kopiowanie z `-E` zgłasza odmowę dostępu lub problem z plikiem AppleDouble `._*`.

**Przyczyna:** Systemowy macOS `openrsync` obsługuje metadane, ACL i xattr przez `-E`; AppleDouble jest sposobem zapisu takich metadanych. Cel lub plik może ich nie przyjąć.

**Bezpieczna diagnostyka:** `rsync -an --checksum <SOURCE_FILE> <DESTINATION>` oraz `rsync -anE --checksum <SOURCE_FILE> <DESTINATION>`.

**Rozwiązanie:** Jeśli wariant bez `-E` działa, dla kodu i projektów użyj `rsync -a`, **o ile** ACL/xattr nie są wymagane. Jeśli są wymagane, ustal dokładnie, które metadane zawodzą i użyj zgodnego celu.

**Weryfikacja:** Checksum dry-run, SHA256 i osobna kontrola wymaganych metadanych; zobacz [weryfikację](#verification).

### 2. `rsync: mkstempsock: Invalid argument`

**Objaw:** `rsync` zatrzymuje się na pliku specjalnym.

**Przyczyna:** Socket UNIX lub inny plik specjalny nie daje się odtworzyć na docelowym systemie plików.

**Bezpieczna diagnostyka:** `find <SOURCE> -type s -print` oraz `find <SOURCE> \( -type s -o -type p -o -type b -o -type c \) -print`.

**Rozwiązanie:** Zidentyfikuj konkretny socket runtime i wyklucz **wyłącznie** tę ścieżkę z kopiowania. Nie wykluczaj całego `.git` ani katalogu projektu.

**Weryfikacja:** Ponów `rsync` z tym samym filtrem i wykonaj checksum dry-run; zobacz [backup](#backup).

### 3. Socket Git fsmonitor

**Objaw:** Kopiowanie repozytorium zatrzymuje się przy `.git/fsmonitor--daemon.ipc`.

**Przyczyna:** To socket IPC działającego procesu, nie trwała część historii repozytorium.

**Bezpieczna diagnostyka:** `find <PROJECT>/.git -type s -print` i kontrola wskazanej ścieżki.

**Rozwiązanie:** Wyklucz konkretny socket runtime; zachowaj `.git`, branche, obiekty i metadane worktree.

**Weryfikacja:** Checksum dry-run z identycznym filtrem oraz testy Git z [rozdziału GitHub](#github).

### 4. Przerwane kopiowanie do `.incoming`

**Objaw:** `rsync` skończył z błędem po skopiowaniu części plików.

**Przyczyna:** Częściowy staging jest naturalnym skutkiem przerwania.

**Bezpieczna diagnostyka:** Sprawdź komunikat błędu, `ls -ld <DEST>` i `rsync -acn --delete --itemize-changes <SOURCE>/ <DEST>/` z właściwymi filtrami.

**Rozwiązanie:** Nie kasuj automatycznie `.incoming`. Popraw filtr lub źródło problemu i uruchom `rsync` ponownie; dokończy brakujące dane.

**Weryfikacja:** Checksum dry-run daje zero różnic, następnie SHA256 i ewentualne testy Git; zobacz [backup](#backup).

### 5. Zsh: `PATH` zawiera nagle jeden plik

**Objaw:** `zsh: command not found: shasum`, `awk` lub `find` po pętli.

**Przyczyna:** W Zsh `path` jest specjalną tablicą powiązaną z `PATH`. Kod `for path in ...` może podmienić ścieżkę programów.

**Bezpieczna diagnostyka:** `print -r -- "$PATH"` i `typeset -p path` w uszkodzonej sesji.

**Rozwiązanie:** Uruchom świeżą sesję: `exec /usr/bin/env -u PATH /bin/zsh -l`. W skryptach używaj nazw `FILE`, `NAME`, `ITEM` zamiast `path`.

**Weryfikacja:** `command -v shasum awk find` wskazuje programy, a pierwotne polecenie działa.

### 6. `zsh: file exists: SHA256SUMS`

**Objaw:** Powtórne utworzenie manifestu nie działa.

**Przyczyna:** Opcja `noclobber` blokuje zwykłe `>` przy istniejącym pliku.

**Bezpieczna diagnostyka:** `setopt | rg noclobber` i `ls -l SHA256SUMS`.

**Rozwiązanie:** Po potwierdzeniu właściwego katalogu wygeneruj manifest poleceniem z `>| SHA256SUMS` z [weryfikacji](#verification). Stary manifest po zmianie plików może dawać fałszywy mismatch.

**Weryfikacja:** `shasum -a 256 -c SHA256SUMS` daje `OK` dla wszystkich pozycji, liczba wpisów się zgadza.

### 7. SHA256: `FAILED open or read` po przeniesieniu

**Objaw:** Plik istnieje w nowej lokalizacji, lecz kontrola manifestu nie może go otworzyć.

**Przyczyna:** Manifest zawiera starą absolutną ścieżkę.

**Bezpieczna diagnostyka:** Odczytaj ścieżkę z manifestu, porównaj zapisany hash z wynikiem `shasum -a 256 <DESTINATION>` dla odpowiadającego pliku. Nie publikuj hashy plików wrażliwych.

**Rozwiązanie:** Jeśli hash zapisany i rzeczywisty są identyczne, przebuduj manifest ze ścieżkami względnymi według [weryfikacji](#verification). Jeśli hashe się różnią, wyjaśnij rozbieżność przed zmianą manifestu.

**Weryfikacja:** Z katalogu snapshotu `shasum -a 256 -c SHA256SUMS` przechodzi po przeniesieniu.

### 8. `rsync` pokazuje `.d..t.... ./`

**Objaw:** Dry-run zgłasza tylko katalog główny mimo zgodnych plików.

**Przyczyna:** Różni się `mtime` katalogu; ten kod nie oznacza różnicy treści plików.

**Bezpieczna diagnostyka:** `rsync -acnO --delete --itemize-changes <SOURCE>/ <DEST>/`.

**Rozwiązanie:** Do porównania treści użyj `-O`, które pomija czas katalogów. Jeśli czas katalogu jest istotny, sprawdź go i popraw osobno po analizie.

**Weryfikacja:** Dry-run z `-O` nie wykazuje różnic plików, a SHA256 przechodzi; zobacz [weryfikację](#verification).

### 9. PlistBuddy: `Delete: Entry ... Does Not Exist`

**Objaw:** Usunięcie wpisu nie działa, mimo że klucz jest w pliku.

**Przyczyna:** Nazwy kluczy ze spacjami lub nawiasami mogą być błędnie interpretowane przez składnię PlistBuddy.

**Bezpieczna diagnostyka:** Odczytaj plist przez Python `plistlib` i wypisz **same nazwy kluczy**, bez wartości mogących zawierać sekrety.

**Rozwiązanie:** Po zachowaniu kopii użyj `plistlib` do zmiany jednego dokładnie wskazanego klucza, zapisz do pliku tymczasowego w tym samym katalogu, zachowaj permissions i dopiero wtedy zastąp plik. Nie kasuj całego plist.

**Weryfikacja:** `plutil -lint <PLIST_FILE>` oraz odczyt konkretnego klucza; zobacz [konfiguracje](#safari).

### 10. Usuwanie aplikacji: `Permission denied`

**Objaw:** Usunięcie aplikacji z `/Applications` jest zablokowane.

**Przyczyna:** Aplikacja może należeć do `root` lub mieć ograniczenia uprawnień i flag.

**Bezpieczna diagnostyka:** `ls -ldOe "/Applications/<APP_NAME>.app"` oraz `stat -f 'owner=%Su group=%Sg mode=%Sp flags=%Sf' "/Applications/<APP_NAME>.app"`.

**Rozwiązanie:** Najpierw potwierdź, że to właściwa aplikacja, że jej dane i konfiguracja są zabezpieczone i że użytkownik świadomie chce ją usunąć. Jeśli właścicielem jest `root` i uprawnienia tego wymagają, użyj `sudo` **tylko** dla konkretnej ścieżki aplikacji. Bez wildcardów.

**Weryfikacja:** Sprawdź brak tej dokładnej aplikacji i działanie pozostałych programów; zobacz [bezpieczeństwo](#security).

### 11. Usunięte rozszerzenie Safari nadal widnieje

**Objaw:** Stary wpis rozszerzenia pozostaje w konfiguracji.

**Przyczyna:** Rejestr rozszerzeń lub plist nadal zawiera identyfikator.

**Bezpieczna diagnostyka:** `pluginkit -m -A -D` i wyszukiwanie `<BUNDLE_IDENTIFIER>` w odpowiednich plistach bez wypisywania wartości wrażliwych.

**Rozwiązanie:** Ustal, czy identyfikator jest martwy; usuń tylko konkretny wpis z właściwego plist po zabezpieczeniu kopii. Nie kasuj całych plistów.

**Weryfikacja:** `plutil -lint <PLIST_FILE>`, ponowny odczyt wpisu i kontrola Safari; zobacz [konfiguracje](#safari).

### 12. Dwie pozycje wyglądają jak dwa rozszerzenia

**Objaw:** System pokazuje aplikację i rozszerzenie osobno.

**Przyczyna:** Jedna aplikacja może zawierać osadzone rozszerzenie:

```text
<APP_NAME>.app
└── Contents/PlugIns/<EXTENSION>.appex
```

**Bezpieczna diagnostyka:** Sprawdź strukturę pakietu aplikacji i identyfikatory w `pluginkit -m -A -D`.

**Rozwiązanie:** Traktuj tę parę jako aplikację i jej embedded extension, jeśli ścieżki to potwierdzają; nie usuwaj jednej pozycji wyłącznie na podstawie liczby wpisów.

**Weryfikacja:** Aplikacja i rozszerzenie działają, a konfiguracja wskazuje oczekiwaną parę; zobacz [Safari](#safari).

### 13. Dwa backupy wyglądają podobnie

**Objaw:** Nie wiadomo, czy starszą kopię można uznać za duplikat.

**Przyczyna:** Nazwy i rozmiary nie dowodzą identycznej treści.

**Bezpieczna diagnostyka:** Oblicz SHA256 odpowiadających sobie plików lub manifesty obu katalogów.

**Rozwiązanie:** Ten sam SHA256 wskazuje duplikat treści pliku; inny SHA256 oznacza unikalną wersję. Przed redukcją całych snapshotów uwzględnij też listę plików, metadane i historię Git.

**Weryfikacja:** Porównanie wszystkich wymaganych hashy, liczby plików i zakresu snapshotów; zobacz [weryfikację](#verification).

### 14. Projekt zajmuje kilka GB przez build i cache

**Objaw:** Backup projektu jest nieproporcjonalnie duży.

**Przyczyna:** Katalogi build, cache lub zależności zawierają dane generowane automatycznie.

**Bezpieczna diagnostyka:** `du -sh <PROJECT>/*` i wyszukiwanie katalogów z [Projektów](#projects).

**Rozwiązanie:** Po audycie dodaj precyzyjne exclude tylko dla danych odtwarzalnych. Gotowy APK, IPA lub inny artefakt może być jedyną kopią i wtedy nie wolno go automatycznie wykluczać.

**Weryfikacja:** Lista pominięć jest uzasadniona, checksum dry-run przechodzi z tymi samymi filtrami, a unikalne artefakty są obecne.

### 15. Lokalna gałąź nie istnieje na GitHub

**Objaw:** `git clone` nie daje gałęzi widocznej lokalnie.

**Przyczyna:** Gałąź lub jej commity nie zostały wypchnięte.

**Bezpieczna diagnostyka:** `git -C <PROJECT> branch -a -vv`, `git -C <PROJECT> worktree list` i `git -C <PROJECT> log --branches --not --remotes --oneline`.

**Rozwiązanie:** Przed redukcją `.git` zachowaj pełne repozytorium i powiązane worktree, jeśli coś istnieje tylko lokalnie.

**Weryfikacja:** W kopii widać te same branche, commity i worktree; zobacz [GitHub](#github).

### 16. `git fsck` pokazuje dangling commit/tree/blob

**Objaw:** W raporcie występuje `dangling commit`, `dangling tree` lub `dangling blob`.

**Przyczyna:** Obiekty nie są osiągalne z aktualnych referencji; nie oznacza to automatycznie uszkodzenia.

**Bezpieczna diagnostyka:** `git -C <PROJECT> fsck --full` i ocena historii oraz referencji.

**Rozwiązanie:** Zachowaj pełne `.git` w historycznym backupie. Nie kasuj dangling objects tylko dlatego, że są dangling; mogą zawierać jedyną wersję danych.

**Weryfikacja:** Kopia przechodzi kontrolę integralności i zachowuje oczekiwane obiekty; zobacz [GitHub](#github).

### 17. Manifest zawiera samego siebie

**Objaw:** Liczba wpisów jest większa niż liczba plików danych albo kontrola manifestu jest niestabilna.

**Przyczyna:** `SHA256SUMS` został uwzględniony przy liczeniu własnego hasha.

**Bezpieczna diagnostyka:** Porównaj liczbę wpisów z liczbą zwykłych plików danych, pomijając `SHA256SUMS`.

**Rozwiązanie:** Po sprawdzeniu katalogu utwórz manifest ponownie:

```zsh
find . -type f ! -name SHA256SUMS -print0 \
  | sort -z \
  | xargs -0 shasum -a 256 >| SHA256SUMS
```

**Weryfikacja:** `shasum -a 256 -c SHA256SUMS` przechodzi, a przy 100 plikach danych jest 100 wpisów i 101 plików łącznie.

### 18. Czy `.incoming` można finalizować?

**Objaw:** Kopia wygląda na gotową, lecz nie ma pewności co do kompletności.

**Przyczyna:** Samo zakończenie kopiowania nie dowodzi zgodności ani poprawności repozytorium.

**Bezpieczna diagnostyka:** Sprawdź kolejno: źródło istnieje; staging istnieje; `rsync` skończył bez błędu; checksum dry-run ma zero różnic; SHA256 ma PASS; dla krytycznych repozytoriów `git fsck` przechodzi; liczba plików się zgadza.

**Rozwiązanie:** Dopiero po spełnieniu checklisty i zabezpieczeniu poprzedniego finalnego stanu wykonaj `mv` z `.incoming` do nazwy finalnej; zobacz [procedurę](#backup).

**Weryfikacja:** Finalny katalog istnieje, nie ma konkurencyjnej częściowej kopii, a manifest przechodzi z nowej lokalizacji.

### 19. Czy stare źródło można usunąć?

**Objaw:** Migracja wygląda na skończoną, ale stare dane zajmują miejsce.

**Przyczyna:** Pozorna zgodność może ukrywać unikalne wersje, branche, klucze lub artefakty.

**Bezpieczna diagnostyka:** Potwierdź istnienie `CURRENT`/`ARCHIVE`, SHA256 PASS, checksum źródło ↔ archiwum PASS, rozdzieloną historię, duplikaty rozpoznane hashami i brak unikalnych branchy, kluczy oraz artefaktów.

**Rozwiązanie:** DELETE wykonaj ostatni, wyłącznie dla dokładnie wskazanego i zweryfikowanego starego źródła, po świadomej decyzji właściciela danych.

**Weryfikacja:** Sprawdź, że `CURRENT` i `ARCHIVE` nadal przechodzą SHA256 oraz że potrzebne dane dają się odtworzyć; zobacz [odtwarzanie](#restore).

### 20. Szybka diagnostyka komunikatów

**Objaw:** Pojawia się jeden z typowych komunikatów poniżej.

**Przyczyna:** Najczęstsze przyczyny zestawiono w tabeli; konkretny przypadek wymaga potwierdzenia.

**Bezpieczna diagnostyka:** Wybierz odpowiedni wiersz i uruchom wskazaną kontrolę przed zapisem.

**Rozwiązanie:** Zastosuj rozwiązanie dopiero po potwierdzeniu przyczyny i warunków z odpowiedniego problemu.

**Weryfikacja:** Powtórz kontrolę z wiersza, a dla kopii wykonaj [checksum i SHA256](#verification).

| Komunikat / objaw | Najczęstsza przyczyna | Co sprawdzić | Rozwiązanie |
| --- | --- | --- | --- |
| `Permission denied` przy `rsync -E` | AppleDouble, ACL lub xattr | Dry-run z `-E` i bez `-E` | Dla kodu bez wymaganych metadanych użyj `-a`; zobacz problem 1. |
| `mkstempsock: Invalid argument` | Socket/runtime | `find <SOURCE> -type s -print` | Wyklucz tylko rozpoznany socket; problemy 2–3. |
| `command not found` po pętli Zsh | Nadpisane `path`/`PATH` | `print -r -- "$PATH"` | Nowa sesja i zmiana nazwy zmiennej; problem 5. |
| `file exists: SHA256SUMS` | `noclobber` | `setopt` i właściwy katalog | `>| SHA256SUMS`; problem 6. |
| `.d..t.... ./` | Czas katalogu | Dry-run z `-O` | Oddziel kontrolę czasu od treści; problem 8. |
| `FAILED open or read` | Stara absolutna ścieżka | Hash zapisany i aktualny | Manifest względny po potwierdzeniu hashy; problem 7. |
| `Delete: Entry Does Not Exist` | Składnia klucza plist | Dokładna nazwa klucza | `plistlib` i jeden wpis; problem 9. |
| `Permission denied` przy usuwaniu aplikacji | Właściciel/flags | `ls -ldOe`, `stat` | Dokładna ścieżka, ewentualnie `sudo`; problem 10. |
| `dangling commit/tree/blob` | Obiekt poza referencjami | `git fsck --full` | Zachowaj obiekty do oceny; problem 16. |
| Checksum dry-run pokazuje różnice | Brak pliku, inna treść lub dodatkowy manifest | Kody `rsync`, filtry i lista plików | Wyjaśnij każdą różnicę, ponów kopiowanie; problemy 4 i 8. |

[↑ Powrót do spisu](#index)

<a id="publishing"></a>
## Co można bezpiecznie opublikować na GitHub

**TAK:** strukturę katalogów, przykładowe komendy, placeholdery, ogólne zasady, przykładowe exclude i troubleshooting bez prywatnych danych.

**NIE:** private keys, tokeny, hasła, credentials, prywatne IP, UUID, osobiste ścieżki, nazwy prywatnych projektów i repozytoriów ani zawartość plików mogących zawierać sekrety. Przed `git add` i publikacją przejrzyj plik oraz wynik skanera sekretów, a przed publicznym push także historię commitów.

Ten plik jest głównym dokumentem publicznym. Jeśli później powstaną dodatkowe publiczne instrukcje, dodaj tu linki do nich; każda niech linkuje z powrotem do `README.md`, a dokumenty powiązane niech linkują między sobą. Nie powielaj całych instrukcji.

[↑ Powrót do spisu](#index)

<a id="example"></a>
## Przykład konfiguracji

**TO TYLKO PRZYKŁAD. Wszystkie dane są fikcyjne.**

```text
<HOME> = /Users/alex
<BACKUP_VOLUME> = /Volumes/Backup
<GITHUB_USERNAME> = alex-dev
<REPOSITORY_NAME> = ExampleApp
<PROJECTS_DIR> = /Users/alex/Projects
<CATEGORY> = Projects
<YYYY-MM-DD> = 2026-01-15
```

Przykładowy wynik po podstawieniu danych pokazuje podział na najnowszy stan i punkt historyczny:

```text
/Volumes/Backup/CURRENT/Projects/ExampleApp/
/Volumes/Backup/ARCHIVE/Projects/2026-01-15/
```

Przed użyciem u siebie zastąp każdą wartość własną, sprawdź montowanie woluminu i przejdź przez [procedurę backupu](#backup).

[↑ Powrót do spisu](#index)
