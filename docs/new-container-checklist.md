---
title: "New Container Onboarding Checklist"
status: active
type: how-to
owner: homelab
last_reviewed: 2026-03-05
---

# New Container Onboarding Checklist

**Last Updated:** 2026-03-05
**Purpose:** Roadmap dla operatora i dla sesji LLM przy dodawaniu nowych kontenerów.

---

## Faza 1 — Przed uruchomieniem (docker-compose.yml)

### Obowiązkowe pola w compose file
- [ ] **`restart: unless-stopped`** — zawsze
- [ ] **`healthcheck`** — obowiązkowe, z `start_period` ≥ 30s
  - Sprawdź co ma kontener: `wget`? `curl`? własny `/healthcheck`? `python3 urllib`?
- [ ] **`deploy.resources.limits`** — `memory` + `cpus`
- [ ] **`logging`** — `json-file`, `max-size: 10m`, `max-file: 3`
- [ ] **`user: '${PUID:-1000}:${PGID:-1000}'`** lub udokumentowany wyjątek root

### Sieć
- [ ] Publiczny web UI → `traefik-public`
- [ ] Backend/baza/API → `homelab-internal`
- [ ] Jeśli na wielu sieciach → `traefik.docker.network=traefik-public` OBOWIĄZKOWY

### Jeśli widoczny przez Traefik
- [ ] `traefik.enable=true`
- [ ] `traefik.docker.network=traefik-public`
- [ ] Router HTTP + Router HTTPS (TLS)
- [ ] Service z poprawnym portem wewnętrznym

### Jeśli to baza danych
- [ ] Rozważ **Tier 1** (ręczna aktualizacja, backup przed każdą zmianą)
- [ ] Healthcheck sprawdza gotowość DB (nie tylko TCP)
- [ ] Brak `user:` dla MySQL/MariaDB (init wymaga root)

---

## Faza 2 — Po uruchomieniu (weryfikacja)

- [ ] `docker ps` → kontener `Up (healthy)`
- [ ] URL dostępny z przeglądarki (HTTPS, brak błędu cert)
- [ ] Logi bez błędów: `docker logs <container> --tail 50`

---

## Faza 3 — Integracja z infrastrukturą (OBOWIĄZKOWE)

### DNS
- [ ] **AdGuard** → Filters → DNS Rewrites → dodaj `name.<homelab-domain> → <HOST-IP>`

### Update Strategy (`scripts/update-strategy.sh`)
- [ ] Wybierz tier:
  - **Tier 1** (manual): bazy danych, DNS, reverse proxy
  - **Tier 2** (semi-auto): aplikacje z danymi użytkownika, password managery
  - **Tier 3** (full auto): stateless apps, monitorning exporters, narzędzia
- [ ] Dodaj do `TIER{N}_SERVICES`, `TIER{N}_HEALTH`, `TIER{N}_COMPOSE_SERVICE`
- [ ] Dodaj do `TIER{N}_ORDER` w odpowiedniej pozycji
- [ ] Zaktualizuj komentarz headerowy (liczba serwisów)

### Dokumentacja
- [ ] **`docs/quick-reference/SERVICES.md`** → nowa sekcja lub wiersz w tabeli
- [ ] **`claude-code-docs.md`** → Quick Reference Card + Service Catalog
- [ ] **`homelab-export.sh`** → statyczne liczby tierów w Section 13

### Uptime Kuma
- [ ] Restart: `docker compose -f <repo-root>/uptime-kuma/docker-compose.yml up -d`
- [ ] Dodaj `extra_hosts` do `uptime-kuma/docker-compose.yml` jeśli nowy domain
- [ ] Uptime Kuma UI → Add Monitor (typ: HTTPS, URL: https://name.<homelab-domain>, 60s)

### Monitoring (opcjonalne — zależnie od serwisu)
- [ ] Prometheus: dodaj scrape target w `prometheus.yml` jeśli eksponuje `/metrics`
- [ ] Alertmanager: dodaj route jeśli potrzeba specjalnego routingu alertów

---

## Faza 4 — Finalizacja

- [ ] `git add <service>/docker-compose.yml && git commit -m "feat(<service>): ..."`
  - **NIE commituj `.env` files!**
- [ ] Zaktualizuj `NEXT_SESSION_TODO.md` jeśli są follow-up taski

---

## Decyzja: który tier?

| Kryterium | Tier 1 | Tier 2 | Tier 3 |
|-----------|--------|--------|--------|
| Typ | Bazy danych, DNS, proxy | Aplikacje z danymi | Stateless, eksportery |
| Aktualizacja | Ręczna (backup first!) | Semi-auto + CVE check | Pełny auto |
| Przykłady | MariaDB, PostgreSQL, MongoDB, AdGuard, Traefik | Vaultwarden, Jellyfin, n8n, home-api | BookStack, Mealie, exporters, home-app |
| Czas aktualizacji | Manualnie, przed sesją | Wtorek 05:00 | Poniedziałek 04:00 |

---

## Root Exception (brak user: 1000)

Dokumentuj TYLKO jeśli konieczne. Dopuszczone wyjątki:
1. **MariaDB/MySQL** — init scripts wymagają root
2. **MongoDB** — init scripts wymagają root
3. **Uptime Kuma** — setpriv wymaga root
4. **Traefik** — bind port 80/443 wymaga root lub CAP_NET_BIND_SERVICE
5. *(pozostałe 4 w PERMISSIONS_STRATEGY.md)*

Format komentarza w compose:
```yaml
# ROOT EXCEPTION: [powód] — analogicznie do [podobny_serwis]
```

---

## Szybki checklist (wersja skrócona — copy-paste)

```
NOWY KONTENER CHECKLIST:
[ ] DNS: AdGuard rewrite name.<homelab-domain> → <HOST-IP>
[ ] compose: healthcheck + resources + logging + restart
[ ] update-strategy.sh: tier + SERVICES/HEALTH/COMPOSE_SERVICE/ORDER + header
[ ] SERVICES.md: nowa tabela/wiersz
[ ] claude-code-docs.md: Quick Reference + Service Catalog
[ ] uptime-kuma: extra_hosts + UI monitor
[ ] homelab-export.sh: tier counts w Known Issues
[ ] git commit (bez .env!)
[ ] NEXT_SESSION_TODO.md: follow-up items
```
