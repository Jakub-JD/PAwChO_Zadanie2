# PAwChO - Zadanie 2
**Autor:** Jakub Fus  
**Platforma CI/CD:** GitHub Actions  
**Skaner Podatności:** Trivy  

## 1. Architektura Potoku CI/CD
Zgodnie z wymaganiami zadania, potok został zaprojektowany tak, aby w pełni automatyzować proces budowania, testowania i publikacji obrazu. Łańcuch realizuje następujące założenia:
* Buduje oprogramowanie z wykorzystaniem środowiska `docker-container` i silnika BuildKit na dwie platformy docelowe: `linux/amd64` oraz `linux/arm64`.
* Skraca czas kompilacji używając zdalnego rejestru pamięci podręcznej zlokalizowanego na DockerHub w trybie `mode=max`.
* Wdraża mechanizm bramki bezpieczeństwa (Security Gate) opartej na analizie CVE.

**Realizacja Testu CVE (Blokowanie wypychania uszkodzonego obrazu):**
Silnik Buildx przy jednoczesnym budowaniu wieloplatformowym nie może wczytać gotowego obrazu bezpośrednio do lokalnego demona Dockera. Wymusiło to stworzenie architektury dwuetapowej. W pierwszym kroku obraz jest kompilowany tylko dla domyślnej architektury i wczytywany lokalnie (flaga `load: true`). Skaner Trivy analizuje ten obraz z rygorystycznym parametrem `exit-code: 1` dla podatności `CRITICAL` oraz `HIGH`. Wykrycie luk zatrzymuje cały potok (Fail) i zapobiega wdrożeniu do środowiska produkcyjnego GHCR. Tylko czysty raport pozwala potokowi przejść do ostatecznego etapu budowy multi-arch i wypchnięcia (push) gotowych manifestów na serwer.

## 2. Zastosowana strategia tagowania (Z uzasadnieniem)
Wdrożono elastyczny mechanizm `docker/metadata-action`, który automatycznie przypisuje metadane w oparciu o wyzwalacze (events). Zdefiniowano dwa niezależne wyzwalacze: na zdarzenie `push` (do gałęzi `main`) oraz `workflow_dispatch` (do testów manualnych).

**Tagowanie obrazu produkcyjnego (GHCR):**
* `type=sha,format=long` – Głównym schematem wersjonowania jest pełen hash SHA rewizji z systemu Git. Gwarantuje to bezpośrednie przypisanie obrazu w rejestrze kontenerów do konkretnej, niezmiennej linii kodu. Ułatwia to testowanie wsteczne i jest standardem w architekturach GitOps. 
* `type=raw,value=latest` – Tag ułatwiający pobranie najnowszej kompilacji przez klientów bez konieczności odpytywania dziennika commitów.

**Tagowanie pamięci cache (DockerHub):**
* Zastosowano strategię tzw. "Branch-based caching" ze wskaźnikiem na gałąź `main` (ref=`jakubjd/weatherapp-cache:main`). Izolacja pamięci podręcznej zapewnia, że ewentualne eksperymentalne kompilacje na pobocznych gałęziach nie doprowadzą do zjawiska zanieczyszczenia cache'u (Cache Poisoning), co spowolniłoby główne kompilacje produkcyjne. Użycie backendu `registry` z parametrem `mode=max` zapisuje wszystkie pośrednie kroki buildera (w tym pobieranie zależności Go), maksymalizując optymalizację czasową.
