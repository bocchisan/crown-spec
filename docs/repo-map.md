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
crown-factory/derive ─► crown-games/<name>     (игры: арифметика адреса)
crown-factory/salt  ─► crown-games/<name>      (игры: соль форм офчейн)
crown-relay ──────────► crown-indexer          (по principal в рантайме, не path-dep)
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
deploy/{testnet,mainnet}.toml    минт USDC (константа программы)
tests/                           out==in, ноль баланса, immutable
```

**Solana фабрика (`crown-factory`)**
```
derive/src/{lib,solana}.rs       крейт crown-derive (шов с индексом, заморожен)
salt/src/{lib,two_outcome,payout_table,stream}.rs   крейт crown-salt (офчейн)
shapes/{two-outcome,payout-table,stream}/solana/    Anchor-программы форм
vectors/                         байтовые фикстуры деривации/соли
deploy/{testnet,mainnet}.toml    минт USDC, splitter, DOMAIN
scripts/                         структурные линты, генератор векторов
```

**Канистра-ядро (`crown-indexer`, `crown-relay`)**
```
src/{lib,api,…}.rs               ic-cdk, ic-stable-structures; build.rs бейкает профиль
config/{testnet,mainnet}.toml    per-chain + замораживаемые константы
dfx.json · <name>.did            .did — единственный не-query (ingest / submit)
```

**Игра (`crown-games/<name>`)**
```
logic/src/{lib,…}.rs             zero-dep крейт: машина состояний + правило вердикта (LOGIC_VERSION)
canister/src/{lib,api,auth,weight,sign,certify}.rs   ic-cdk; ingress, пруфы, per-scope подпись
config/{testnet,mainnet}.toml
e2e/                             драйвер devnet-транзакций
dfx.json · <name>.did · docs/{spec,build-plan}.md
```

Исключение — `subscription`: детерминированная рассрочка на форме `stream`, **без канистры/логики/резолвера**
— только `docs/` + `e2e/` (вся ончейн-логика в форме `stream`, `crown-factory`).

**Мета (`crown-spec`)** — кода нет: `CLAUDE.md`, `docs/`, `tools/`.

---

## 3. Добавить новый реп (обычно — игру)

1. `crown-games/<name>/` по скелету §2 (тип «Игра»); `CLAUDE.md` → канон + `games-harness.md`.
2. `docs/spec.md` определяет: область/`scope_id`, форму фабрики, список `update`, машину+вердикт,
   прообраз `id`, тайминги (по `games-harness.md` §«что обязана определить спека»).
3. `docs/build-plan.md` — свои этапы + DoD (вкл. инварианты не-отрицательности `cost.md §6`).
4. Ложится на готовую форму `two-outcome`/`payout-table`/`stream` → **периметр не трогается**; строка в
   allowlist релея. Нужна новая форма → новая фабрика = отдельное решение (`08-deferred.md`).

Ядро при этом неизменно: карта §1 растёт только в блоке `crown-games/`.
