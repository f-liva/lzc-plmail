# lzc-plmail

[plMail](https://github.com/karatektus/pl_mail) impacchettato come `.lpk` per
il NAS Lazycat. Nessuna immagine nostra: il wrapper è solo il manifest, perché
l'immagine upstream `ghcr.io/karatektus/pl_mail` fa già tutto da sé — genera i
propri segreti, migra il database e supervisiona i processi di background.

plMail è un **client** di posta, non un server: si collega alle caselle che hai
già (IMAP, Gmail, Outlook), le sincronizza in un Postgres tuo e le mostra in
un'unica inbox, con un calendario accanto. Non riceve posta da internet e non
ospita domini, quindi niente porta 25 da aprire.

- URL pubblico: <https://plmail.federicoliva.it> (tunnel Cloudflare + Access)
- URL interno: <https://plmail.chronos.heiyu.space> (gateway Lazycat)
- Package: `cloud.lazycat.app.plmail`

## Architettura

```
                 ┌─ Cloudflare Access ─┐
browser ──┬─────>│ plmail.federicoliva.it  (tunnel, policy: 2 email)
          └─────>│ plmail.chronos.heiyu.space (gateway Lazycat, login microserver)
                 └──────────┬──────────┘
                            │
                            └─> php (FrankenPHP :80, Caddy in chiaro)
                                  ├── /.well-known/mercure* ──> worker (hub Mercure :80)
                                  └── Postgres ──────────────> database
```

L'app risponde su entrambi gli hostname. `APP_PUBLIC_URL` punta a quello
pubblico, perché è l'host che finisce nei link assoluti delle notifiche push e
`plmail.chronos.heiyu.space` non risolve fuori casa.

Tre service, tutti e tre con l'immagine upstream pinnata a digest:

| Service | Cosa esegue |
|---|---|
| `php` | FrankenPHP che serve l'app |
| `worker` | `app:work`: hub Mercure, supervisore IMAP IDLE, quattro consumer Messenger |
| `database` | `postgres:18-alpine` |

## Divergenze dal `compose.yaml` upstream, e perché

**Niente `secrets-init`.** Quel container one-shot esiste perché l'immagine
Postgres legge la password alla *creazione* del container, prima che l'app
possa generargliela — e Lazycat non ha l'equivalente di
`condition: service_completed_successfully`. Qui la password Postgres sta nel
manifest e viaggia dentro `DATABASE_URL`: `generate-secrets` riconosce un DSN
che porta già una password e non ne genera una seconda (vedi
`database_url_has_password` in `frankenphp/generate-secrets.sh` upstream). Gli
altri tre segreti — `APP_SECRET`, `APP_ENCRYPTION_KEY`, `MERCURE_JWT_SECRET` —
li genera l'entrypoint di `php` e `worker` sotto `flock`, nella directory che i
due condividono.

**Un solo bind per i dati.** È il pattern NAS della doc upstream:
`APP_STORAGE_DIR=var/data` raccoglie allegati, messaggi grezzi, upload *e* i
segreti sotto `/lzcapp/var/data`. Una cosa sola da backuppare, e i due service
che vedono dimostrabilmente la stessa chiave di cifratura.

**`SERVER_NAME=:80`.** Il TLS lo termina il proxy davanti — il gateway Lazycat
sul dominio interno, Cloudflare su quello pubblico — quindi Caddy dentro il
container serve HTTP in chiaro e non prova a ottenere un certificato per un
nome che non controlla.

**`MERCURE_UPSTREAM`.** Il Caddyfile dell'immagine inoltra
`/.well-known/mercure*` all'host `mercure`, che nel compose upstream è un alias
di rete del worker. Gli alias di rete qui non ci sono: si punta al nome lzcapp
del worker, che è il container che esegue l'hub (su `:80`, vedi
`BackgroundProcesses::HUB`).

**Immagine a digest, non a tag di versione.** Upstream pubblica su ghcr solo
`latest`, `main` e `sha-<commit>`: un `:v0.3.6` non esiste e il pull muore con
`manifest unknown` (e Lazycat lo riporta solo in
`/lzcsys/data/system/pkgm/run/<pkg>/errmsg`, non in `app log`). Il digest nel
manifest è `latest` al 20/10/2026, cioè `APP_VERSION` v0.3.6, commit
`aba0130`.

## Due cose da backuppare, separatamente

`/lzcsys/data/appvar/cloud.lazycat.app.plmail/`:

- `postgres/` — il database, dove sta la posta;
- `data/secrets/generated.env` — la chiave che apre le credenziali dentro il
  database.

Tenerli separati è il punto: perdere `APP_ENCRYPTION_KEY` rende le password
delle caselle salvate **illeggibili per sempre**, e nessun ripristino del solo
database le recupera.

## Build e install

```bash
cd plmail-lzc/
lzc-cli project build
cp cloud.lazycat.app.plmail-v*.lpk /mnt/c/Users/fede9/Desktop/
powershell.exe -Command "lzc-cli app install C:\Users\fede9\Desktop\cloud.lazycat.app.plmail-v1.0.1.lpk"
```

## Aggiornare

Upstream rilascia spesso (0.3.2 → 0.3.6 nello stesso giorno). Per salire di
versione:

```bash
TOK=$(curl -s "https://ghcr.io/token?scope=repository:karatektus/pl_mail:pull&service=ghcr.io" \
  | python3 -c "import sys,json;print(json.load(sys.stdin)['token'])")
curl -sI -H "Authorization: Bearer $TOK" \
  -H "Accept: application/vnd.oci.image.index.v1+json" \
  https://ghcr.io/v2/karatektus/pl_mail/manifests/latest \
  | grep -i docker-content-digest
```

Si sostituisce il digest in `lzc-manifest.yml`, si bumpa `version:` in
`package.yml` (l'orchestrator riconosce il bump semver come nuova versione e
ri-pulla l'immagine), si rifà il build e si reinstalla. Le migrazioni del
database girano da sé al boot, sotto advisory lock Postgres.

## Diagnosi

```bash
# container
lzc-docker ps -a --format "{{.Names}} | {{.Status}}" | grep plmail

# log — `lzc-cli app log` risponde "not yet realized", usare questi
lzc-docker logs --tail 50 cloudlazycatappplmail-php-1
lzc-docker logs --tail 50 cloudlazycatappplmail-worker-1

# un install che non parte: l'errore sta QUI, non nei log
tail -c 2000 /lzcsys/data/system/pkgm/run/cloud.lazycat.app.plmail/errmsg

# l'app risponde? (dal container, bypassando il gateway)
lzc-docker exec cloudlazycatappplmail-php-1 \
  curl -s -o /dev/null -w '%{http_code}\n' http://localhost/healthz
```

## Licenza

Wrapper MIT. plMail è AGPL-3.0, di
[karatektus](https://github.com/karatektus/pl_mail).

## Gotcha: plMail si termina da solo, e Lazycat non lo riavvia

`/install`, il reset dei dati e le azioni di sistema dell'admin finiscono
chiamando `WebProcessRestart`, che manda `SIGTERM` a PID 1 per far ripartire il
web process con la chiave di cifratura appena generata
(`src/Service/Monitoring/WebProcessRestart.php`). Upstream da' per scontato un
orchestrator che lo tiri su di nuovo.

L'orchestrator Lazycat **scarta** `services.*.restart`: il lint di build lo
segnala come campo ignoto, e il `docker-compose.yml` generato non lo contiene.
Provato il 20/10/2026 con la 1.0.3 — policy effettiva ancora `no`. Quindi php
esce con 0 e resta giu'.

Nel browser il sintomo e' *"That didn't work — something went wrong on our
side"* **dopo** che l'operazione e' riuscita: il POST risponde 200, poi il
backend non c'e' piu'. Il primo amministratore nel nostro caso era stato creato
regolarmente.

Recupero, dal NAS:

```bash
lzc-docker start cloudlazycatappplmail-php-1
```

Al boot del microserver ci pensa `pkgm`, che ricrea i container.
