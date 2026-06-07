# PAwChO - Zadanie 2
**Autor:** Jakub Fus  
**Platforma CI/CD:** GitHub Actions  
**Skaner Podatności:** Trivy  

## 1. Jak działa mój potok CI/CD?
W tym repozytorium znajduje się rozwiązanie zadania 2. Zrobiłem automatyczny potok w GitHub Actions, który po każdym pushu ogarnia za mnie całą robotę z obrazami. Główne założenia:
* Obraz buduje się na dwie architektury (`linux/amd64` i `linux/arm64`) przy użyciu rozszerzenia BuildKit.
* Żeby nie tracić czasu, potok korzysta z cache'a zapisywanego w moim publicznym repo na DockerHubie (backend `registry` w trybie `mode=max`).
* Dodałem skaner Trivy, żeby upewnić się, że obraz nie ma poważnych dziur przed wysłaniem w świat.

**Rozwiązanie warunku (c) - Test bezpieczeństwa CVE:**
Zwykły Docker Buildx nie potrafi jednocześnie budować obrazu na kilka architektur i wrzucać go od razu do lokalnego Dockera. Dlatego musiałem to rozbić na dwa etapy. 
Najpierw potok buduje obraz tylko lokalnie (korzystając z flagi `load: true`). Potem odpala się skaner Trivy, który bierze ten obraz pod lupę pod kątem podatności `CRITICAL` oraz `HIGH`. Ustawiłem mu parametr `exit-code: 1`, więc jeśli znajdzie jakieś poważne luki, po prostu "wywala" potok i przerywa całą akcję. Dopiero kiedy test przejdzie na zielono, odpala się ostateczne budowanie multi-arch i czysty obraz leci na mojego publicznego GitHuba (GHCR).

## 2. Przyjęty schemat tagowania (Uzasadnienie)
Do generowania tagów użyłem gotowej akcji `docker/metadata-action`. Całość uruchamia się po pushu na gałąź `main` albo przy ręcznym odpaleniu (`workflow_dispatch`).

**Tagowanie gotowego obrazu (GHCR):**
* `type=sha,format=long` – Głównym tagiem obrazu jest długi hash commita z GitHuba (np. `sha-5413...`). Użyłem tego schematu, bo to standard w podejściu GitOps. Dzięki temu zawsze wiem, z jakiej dokładnie wersji kodu powstał dany obraz i łatwo mogę do tego wrócić.
* `type=raw,value=latest` – Zwykły tag `latest` wrzucony po prostu dla wygody, żeby ułatwić pobieranie najnowszej paczki.

**Tagowanie pamięci cache (DockerHub):**
* Do cache'owania przyjąłem strategię opartą na nazwach gałęzi (mój tag to po prostu `main`). Wybrałem takie rozwiązanie, aby odizolować główny cache od ewentualnych eksperymentów na innych branchach. Gdybym wrzucał wszystko do jednego tagu, ryzykowałbym tzw. zanieczyszczeniem cache'u (Cache Poisoning) przez zepsuty kod. Dodatkowo używam trybu `mode=max`, który eksportuje wszystkie warstwy pośrednie buildera (nawet pobieranie paczek do Go), co daje potężnego "kopa" do prędkości przy kolejnym budowaniu.
* Zastosowano strategię tzw. "Branch-based caching" ze wskaźnikiem na gałąź `main` (ref=`jakubjd/weatherapp-cache:main`). Izolacja pamięci podręcznej zapewnia, że ewentualne eksperymentalne kompilacje na pobocznych gałęziach nie doprowadzą do zjawiska zanieczyszczenia cache'u (Cache Poisoning), co spowolniłoby główne kompilacje produkcyjne. Użycie backendu `registry` z parametrem `mode=max` zapisuje wszystkie pośrednie kroki buildera (w tym pobieranie zależności Go), maksymalizując optymalizację czasową.
