# Crown — План сборки

Единственный источник **стадии проекта**. В начале сессии Claude читает этот файл и определяет текущий
этап — первый не-`DONE`. Снизу вверх; один этап = один связный проверяемый кусок; дальше не идти, пока
DoD не выполнен и тесты зелёные.

Всё на devnet до последнего этапа; передеплой свободен. Reproducible build доверенного периметра —
на каждом его этапе.

## Статус

| Этап | Реп(ы) | Что | Статус |
|---|---|---|---|
| P0 | все | Скелеты, CI, структурные линты (границы `01-standards`) | DONE |
| P1 | `crown-reduce` | Чистая свёртка + property-тесты закона | DONE |
| P2 | `crown-splitter` | `donate`, ноль баланса, event-CPI; devnet; authority `--final` | DONE¹ |
| P3 | `crown-factory` | `derive`/`salt`; форма `two-outcome` (root-verdict) + конвенция заголовка; devnet | DONE² |
| P4 | `crown-indexer` | Платный `ingest` (сеттлменты+рождения), keyed-Merkle дерево, пруфы, кэп RPC | DONE³ |
| P5 | `crown-relay` | Фронтер, правило подписи, per-key cap, `inspect_message` | DONE⁴ |
| P6 | `crown-games/*` | Харнесс + `conditional-tasks` (эталон); без-наружу, без-таймеров, ленивое создание, оплаченный pull, голос в `inspect_message` | TODO |
| P7 | `crown-factory`, `crown-games/*` | Формы `payout-table`/`stream`; `funding`/`auction`/`subscription` | TODO |
| P8 | все | **Заморозка + mainnet** | TODO |

¹ P2: тело `donate` (`transfer_checked` + event-CPI), минт-константа из `deploy/` (`build.rs`), SBF-сборка
и litesvm-тесты (out==in / нулевой донат ревертится / донор структурен) — готовы и зелёные. Реальный
devnet-деплой и `authority --final` — операционные шаги (нужен профинансированный кошелёк; сжигание
authority — действие заморозки/P8, на devnet передеплой свободен). Живой devnet-прогон сплиттера —
осознанно в составе **e2e игр (T5/F5/A5)**, где стек деплоится целиком; отдельно поштучно не гоняем. **Тулчейн:** Agave ≥3.1.14 (rustc ≥1.85)
— требуется для SBF-сборки solana 2.3-зависимостей (edition2024); пиннится на reproducible-build (P8).

⁴ P5: `crown-relay` — чистый слой (unit): бейк конфига (index/allowlist/prices/floor/rate-limit) +
`admit`-гейт (allowlist → per-key rate-window → бюджет-флор, инв. #6). Канистра: `submit` (async;
gate → `Call::with_cycles` форвард `Settlement`/`Birth`→`index.ingest`, `Sign`→`game.request_signature`,
raw-байты, релей — тупой прокси над книгой не властен), `inspect_message` (allowlist+размер, отсекает до
исполнения), `init(opt InitArgs)` с override index/allowlist на testnet и **trap на mainnet**, `get_index`.
**e2e (PocketIC 13.0.0, mock-downstream):** не-allowlisted → reject `inspect_message` без форварда;
allowlisted `Settlement`→форвард в index с `INGEST_PRICE`; `Sign`→`request_signature` с `SIGN_PRICE`;
per-key rate-limit капит ключ; low-budget → `LowBudget` без форварда; mainnet-override → trap при install.
`.did` сверен. Правило подписи (`scope_cost`) — политика бэкенда/игры (P6), релей лишь оплачивает.

³ P4: `crown-indexer` — чистый слой (unit): свёртка+keyed-Merkle (`combined_root =
fork_hash(labeled_hash("book"), labeled_hash("births"))`, свидетель реконструирует прямо в корень),
exactly-once (пустая sig не applied), атрибуция §4, распознавание (чужой `Settled` не засчитан,
перекрёстная сверка event↔`TransferChecked`, PDA-редерайв рождения), парсинг wire-формата Solana
(legacy+v0, валидирован против `solana-program`), ingest-gate (порядок оплаты), бейк конфига. Канистра:
async `ingest` (gate→`msg_cycles_accept`→outcall SOL RPC-канистре с кэпом `RESPONSE_MAX_BYTES` и
`ATTACH_CYCLES`→fold→`certified_data_set`), query со свидетелями+`get_certificate`. **e2e (PocketIC,
сервер 13.0.0):** неоплаченный `ingest`→`Underpaid` без outcall (инв. #8); оплаченный (mock-релей с
циклами + mock SOL RPC на пиннутом принципале)→`Applied`→`get_reputation` +gross; дубль не
дублирует; кэп несётся на outcall (инв. #1); сертификат коммитит `certified_data == combined_root`
(lookup в дереве против NNS-инстанса; BLS/делегация — на стандартном клиенте). Persistence —
**in-memory** (вариант A: пересчёт-из-чейна `00 §8`; на mainnet blackhole). `.did` сверен
`candid-extractor`'ом. **Тулчейн:** `pocket-ic` из комплекта dfx (`POCKET_IC_BIN`).

² P3: `crown-derive` (реальный `find_program_address` = SHA256 + off-curve, побитово == Solana),
`crown-salt::two_outcome` (байт-точный хеш), форма `two-outcome` (`create_escrow` с ончейн-солью =
`crown-salt`, `claim` root-verdict + `assert_resolver_signed` ed25519-интроспекцией, `refund` по
`deadline`, конвенция заголовка) — реализованы; SBF-сборка + litesvm-тесты зелёные (create/refund/header/
вектор соль↔форма↔адрес, claim `cancel`/подделка подписи). Полный **settle**-путь (форма→сплиттер) —
на devnet e2e (T5): под litesvm 0.7 sysvar инструкций портит слот `account_keys`, когда рядом две
CPI-программы (splitter+token); `settle`-тест помечен `#[ignore]` с пояснением. Логика `settle`
реализована и компилируется.

## Ключевые DoD по этапам

- **P0** — три линта на пустых репах: `cargo tree -p crown-reduce` = один крейт; grep-границы пусты;
  mainnet-профиль без `Custom`. Дерево репозиториев = карта `../CLAUDE.md`.
- **P1..P7** — DoD и тесты в спеке своего репа + `01-standards §Тесты`. P4/P5/P6 обязаны закрыть
  инварианты не-отрицательности (`cost.md §6`, тест на каждый): кэп RPC (P4), правило подписи + per-key
  cap (P5), голос-в-`inspect_message` + ленивое создание + ленивая сертификация (P6).
- **P8 — pre-freeze чеклист (до любой заморозки):**
  - [ ] cost-gate: замер `getTransaction`/`sign_with_schnorr`/`getAccountInfo` → `INGEST_PRICE`/
        `SIGN_PRICE`/`ATTACH_CYCLES`/`MIN_GROSS`/`V_MAX·v`/`MARGIN` внесены в `config/` с запасом.
  - [ ] `INGEST_PRICE` ≥ worst-case outcall с кэпом `max_response_bytes`.
  - [ ] Флор выровнен: игровой/фабричный `min_gross` ≥ индексного `MIN_GROSS`.
  - [ ] Дерево + формат свидетеля (книга/рождения) финализированы (после заморозки неисправимы).
  - [ ] Векторы деривации `id` финализированы (конфиг/тайминги в прообразе).
  - [ ] Reproducible build индексатора: хэш wasm воспроизводится третьим лицом.
  - [ ] Механизм пополнения циклов протестирован **до** снятия контроллера (порядок необратим).
  - [ ] `crown-splitter`/`crown-factory`: authority `--final`. `crown-indexer`: контроллер снят.

Новая форма эскроу после заморозки = новая фабрика + новое поколение индекса (`00-architecture §8`);
это не этап здесь, а `08-deferred.md`.
