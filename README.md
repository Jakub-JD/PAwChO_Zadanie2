# PAwChO - Zadanie 2
**Autor:** Jakub Fus  
**Platforma CI/CD:** GitHub Actions  
**Skaner Podatności:** Trivy  

## 1. Architektura Potoku CI/CD
Opracowany łańcuch GitHub Actions w pełni automatyzuje proces budowania i publikacji obrazu. Główne założenia działania:
* Obraz budowany jest z użyciem środowiska `docker-container` na dwie architektury: `linux/amd64` oraz `linux/arm64`.
* W celu optymalizacji czasu budowania, potok wykorzystuje zewnętrzny cache zapisywany w moim publicznym repozytorium na platformie DockerHub (w trybie `mode=max`).
* Zaimplementowano obowiązkowy test bezpieczeństwa (skaner Trivy), który blokuje wysłanie podatnego obrazu do chmury.

**Realizacja testu CVE:**
Ponieważ jednoczesne budowanie obrazu na wiele architektur uniemożliwia jego proste załadowanie do lokalnego demona Dockera, proces podzieliłem na dwa kroki. Najpierw obraz buduje się tylko lokalnie (flaga `load: true`). Następnie skaner Trivy weryfikuje go pod kątem luk bezpieczeństwa. Użyłem parametru `exit-code: 1` dla podatności `CRITICAL` oraz `HIGH`. Oznacza to, że wykrycie poważnych błędów automatycznie przerywa działanie potoku (status Fail) i blokuje wysłanie obrazu. Dopiero gdy test przejdzie pomyślnie, wykonywane jest docelowe budowanie multi-arch i wypchnięcie paczki na serwer GHCR.

## 2. Przyjęty schemat tagowania (Uzasadnienie)
Do zarządzania metadanymi wykorzystałem akcję `docker/metadata-action`. Tagi generowane są automatycznie na podstawie wyzwalacza (uruchomienie po zdarzeniu `push` na gałąź `main` lub ręczne `workflow_dispatch`).

**Tagowanie obrazu produkcyjnego (GHCR):**
* `type=sha,format=long` – Podstawowym tagiem obrazu jest długi hash commita z systemu Git. Takie podejście gwarantuje, że dany obraz jest trwale i jednoznacznie powiązany z konkretną wersją kodu źródłowego. Ułatwia to testowanie i jest zalecane w architekturach GitOps.
* `type=raw,value=latest` – Dodatkowy tag nakładany w celu ułatwienia pobierania najnowszej wersji obrazu przez użytkowników.

**Tagowanie pamięci cache (DockerHub):**
* Zastosowałem tagowanie oparte na nazwie gałęzi (w tym przypadku `main`). Takie rozwiązanie izoluje główny cache od ewentualnych eksperymentów na innych gałęziach. Zapobiega to zjawisku zanieczyszczenia pamięci podręcznej wadliwym kodem. Użycie trybu `mode=max` sprawia, że eksportowane są wszystkie warstwy pośrednie, co dodatkowo przyspiesza kolejne kompilacje.
