# Crown — Карта репозиториев и стандарт раскладки

Одна вещь — один реп; доверенный периметр собирается независимо в тот же хэш. Все репы лежат **рядом**
(sibling-клоны) под корнем воркспейса; кросс-репо ссылки — по имени соседа (`crown-spec/docs/…`).

---

## 1. Карта

```
<workspace>/
├─ crown-spec/            мета: закон, стандарты, стоимость, план, карта — кода нет
├─ crown-reduce/          крейт: чистая свёртка книги (zero-dep, публичный)
├─ crown-splitter/        Solana: программа делителя (immutable)
├─ crown-factory/         Solana: фабрика+эскроу (3 формы) + крейты derive/salt (immutable)
├─ crown-indexer/         канистра ICP: единственный платный индекс (blackhole)
├─ crown-relay/           канистра ICP: фронтер оплаченных вызовов (не заморожен)
└─ crown-games/
   ├─ common/             крейт crown-games-common: всё, что у игр общее и обязано совпадать —
   │                      BLS-пруф, threshold-резолвер, PDA-адрес эскроу, wire-парс,
   │                      вердикт-сообщение, кеш корней, хранилище подписи (инвариант #5),
   │                      `chain_id` (ключ книги) и чтение config/ на бейке
   ├─ e2e-fixtures/       relay-proxy + mock-sol-rpc: тестовые фикстуры, общие для всех игр
   ├─ conditional-tasks/  канистра ICP: игра (эталон)
   ├─ conditional-funding/
   ├─ auction/
   └─ subscription/       + любые будущие игры
```

### Граф зависимостей (path-deps, строго в одну сторону)

```
crown-reduce ─────────────┐
                          ▼
crown-factory/derive ─► crown-indexer          (индекс: свёртка + арифметика адреса)
crown-factory/derive ─► crown-games/common ─► crown-games/<name>   (крипта + арифметика адреса)
crown-factory/salt  ─► crown-games/<name>      (игры: соль форм офчейн)
crown-relay ──────────► crown-indexer          (по principal в рантайме, не path-dep)
crown-relay ──────────► crown-games/<name>     (Sign-форвард; principal из allowlist `games`)
```

`crown-reduce` не знает ни про кого. `crown-splitter` ни от кого не зависит. Обратных зависимостей нет.

### Доверенный периметр и заморозка

| Реп | Роль | Держит деньги/ключи | После mainnet |
|---|---|---|---|
| `crown-splitter` | корень признания №1 | нет / нет | **immutable** |
| `crown-factory` | корень признания №2, кастодия до расчёта | эскроу — до расчёта | **immutable** |
| `crown-indexer` | книга (derived) | нет / нет | **blackhole** |
| `crown-reduce` | закон свёртки | нет / нет | публичный, версионирован |
| `crown-relay` | оплата вписей/подписей | бюджет циклов | **не заморожен** (управляется) |
| `crown-games/common` | общая крипта игр | нет / нет | **свободен** (версионируется с играми) |
| `crown-games/*` | логика игр | ключи резолвера | **свободны** (заменяются) |
| `crown-spec` | доки | нет / нет | — |

Порядок сборки (P0–P8) и pre-freeze чеклист — `07-build-plan.md`. Архитектура — `00-architecture.md`.

---

## 2. Стандарт раскладки репа

Общий скелет — во **всех** репах:

```
<repo>/
├─ CLAUDE.md          тонкий: одна строка «что это» + ссылка на канон + специфика (NEVER-специфика репа)
├─ docs/
│  └─ spec.md         спека репа (что и как; ссылается на 00-architecture / 01-standards / cost)
├─ .github/           CI: fmt, clippy, test + структурные линты (01-standards §Границы)
├─ .gitignore
└─ <код по типу репа, ниже>
```

Исключения из скелета: `crown-spec` — кода нет, поэтому нет ни `.github/`, ни `docs/spec.md`;
`crown-games/common` — внутренний крейт под играми, живёт без своего `CLAUDE.md`/`docs/`/CI (его
проверяет CI каждой игры, которая его тянет).

`build-plan.md` — **центральный** для периметра (`07-build-plan.md`, P0–P8: делаем раз), **свой** у каждой
игры (`docs/build-plan.md`: игры добавляем часто). `config/{testnet,mainnet}.toml` — в каждом сетевом
репе (канистры, программы); значения только там, не в коде (`01-standards §Конфиг`).

### По типу репа

**Крейт (`crown-reduce`)**
```
src/{lib,event,book,reduce}.rs   zero-dep, #![forbid(unsafe_code)]
Cargo.toml                       ноль зависимостей (кроме dev proptest)
```

**Solana-программа (`crown-splitter`)**
```
programs/splitter/src/lib.rs     Anchor, одна инструкция
programs/splitter/build.rs       печёт константы деплоя в код
programs/splitter/tests/         out==in, самосеттлмент, донор структурен, ноль баланса
deploy/{testnet,mainnet}.toml    минт USDC + program id (константы программы)
```

**Solana фабрика (`crown-factory`)**
```
derive/src/{lib,solana}.rs       крейт crown-derive (шов с индексом, заморожен)
salt/src/{lib,two_outcome,stream}.rs   крейт crown-salt (офчейн)
escrow/src/lib.rs                крейт crown-escrow: общие примитивы форм — разбор
                                 ed25519-инструкции и drain-and-close vault (одна копия
                                 критичной логики на two-outcome и stream; домена не знает)
shapes/{two-outcome,stream}/solana/    Anchor-программы форм
deploy/{testnet,mainnet}.toml    минт USDC, splitter, DOMAIN форм
```
Байтовые фикстуры деривации/соли лежат тестами рядом с исходником (`salt/src/*.rs`,
`derive/tests/vectors.rs`), отдельной `vectors/` нет; структурные линты — в `.github/workflows/ci.yml`,
отдельной `scripts/` нет.

**Канистра-ядро (`crown-indexer`, `crown-relay`)**
```
индекс: src/{lib,gate,recognize,parse,rpc,state,certified,config}.rs
релей:  src/{lib,admit,config}.rs
                                 ic-cdk; build.rs бейкает профиль. Стабильной памяти нет:
                                 всё состояние в heap (`crown-indexer/docs/spec.md §Ёмкость`)
config/{testnet,mainnet}.toml    per-chain + замораживаемые константы
e2e-mock/                        мок-даунстрим для PocketIC-тестов
dfx.json · <name>.did            .did — единственный не-query (ingest / submit)
```

**Игра (`crown-games/<name>`)**
```
logic/src/{lib,machine,verdict,…}.rs   zero-dep крейт: машина + правило вердикта (LOGIC_VERSION)
canister/src/{lib,protocol,state,validate,config}.rs   ic-cdk; ingress, пруфы, per-scope подпись
                                 (общая крипта — не здесь, а в crown-games-common)
canister/tests/                  PocketIC: endpoint_e2e + full_e2e (+ birth_e2e у эталона)
config/{testnet,mainnet}.toml    два профиля, третьего нет: имя threshold-ключа задаёт
                                 окружение (key_1 локально и на IC mainnet), а не профиль
e2e/<t5|a5|f5>/                  живой devnet-драйвер игры — отдельный крейт с пустым
                                 [workspace]. Фикстуры (relay-proxy, mock-sol-rpc) — не здесь,
                                 а в crown-games/e2e-fixtures/: они про то, чего не умеет
                                 тестовая среда, и во всех играх одинаковы (P7.6)
dfx.json · <name>.did · docs/{spec,build-plan}.md
```

Исключение — `subscription`: детерминированная рассрочка на форме `stream`, **без канистры/логики/резолвера**
— только `docs/` + `e2e/` (вся ончейн-логика в форме `stream`, `crown-factory`).

**Мета (`crown-spec`)** — кода нет: `CLAUDE.md`, `docs/`, `tools/` (`cost-model.html` — живой
калькулятор стоимости, `cost.md §9`).

---

## 3. Добавить новый реп (обычно — игру)

1. `crown-games/<name>/` по скелету §2 (тип «Игра»); `CLAUDE.md` → канон + `games-harness.md`.
2. `docs/spec.md` определяет: область/`scope_id`, форму фабрики, список `update`, машину+вердикт,
   прообраз `id`, тайминги (по `games-harness.md` §«что обязана определить спека»).
3. `docs/build-plan.md` — свои этапы + DoD (вкл. инварианты не-отрицательности `cost.md §6`).
4. Ложится на готовую форму `two-outcome`/`stream` → **периметр не трогается**; строка в
   allowlist релея. Нужна новая форма → новая фабрика = отдельное решение (`08-deferred.md`).

Ядро при этом неизменно: карта §1 растёт только в блоке `crown-games/`.
