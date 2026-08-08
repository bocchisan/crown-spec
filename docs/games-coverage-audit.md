# Аудит покрытия игр (сценарий → статус)

Дата: 2026-07-27 (пересобрано после правок других сессий). Только аудит — тестов не добавлял,
кода/спек не менял. Цель: полный чеклист всех сценариев по 4 играм с текущим статусом покрытия,
чтобы приоритезировать написание тестов.

> **Поправка 2026-08-02 — закрыт settle-путь обеих MVP-игр.** Этот проход, в отличие от прежних,
> тесты **добавлял**; блок дописан, снимки ниже не переписаны (та же причина, что у `P7.14`/`P7.18`).
>
> **Что было дырой.** Ни один тест не исполнял `accept`/`ready`/`vote` **эндпоинтами** у
> `conditional-tasks`: юниты зовут `state::` напрямую, `full_e2e` шёл `register → decline`, а живой
> драйвер `T5` — тоже только `decline`. То есть ветка «работу приняли → деньги ушли получателю» не
> исполнялась ни в CI, ни на devnet, при том что после переворота умолчания (`LOGIC_VERSION` 5/4) она
> стала веткой **по умолчанию**. У сбора та же ветка жила только в `F5` — ручном devnet-драйвере,
> которого в CI нет.
>
> **Что добавлено** (всё в PocketIC, без денег и без devnet):
>
> - `conditional-tasks/canister/tests/full_e2e.rs::a_weighted_vote_decides_both_verdicts_and_signs_them`
>   и `conditional-funding/…::a_quorate_vote_decides_a_collection_both_ways` — полный путь
>   `create/register → accept → ready → vote → окно закрылось → request_signature`, обе игры, **оба**
>   исхода. Вес голоса — настоящая репутация книги: тест инжектит `Settled` сплиттера через мок SOL RPC,
>   индекс сворачивает его, `get_reputation` отдаёт свидетеля, и голос проходит `inspect_message`
>   обходом хеш-дерева. Подпись каждого исхода проверяется против резолвера — тот же счёт, что делает
>   ончейн-`claim`.
> - **Две области, а не одна, и это существенно.** С перевёрнутым умолчанием одиночный settle-кейс
>   вакуумно-зелёный: тишина тоже даёт `settle`, поэтому он прошёл бы и с непрочитанным голосом.
>   Вторая область голосуется `not_done` тем же голосующим и обязана дать `Cancel`/`Refund`. Проверено
>   мутацией: подмена ожидаемого исхода красит тест (`got DecidedCancel` / `got DecidedRefund`).
> - **Граница** (`inspect_message`), обе игры: рез `MAX_ARG_BYTES` — тем же валидным запросом, добитым
>   неподписанным полем до 9 KiB (непадженный близнец рядом проходит, так что режет именно размер;
>   мутацией подтверждено); повторный `bootstrap` отбит после взятия ключа; байт-в-байт реплей
>   обречённого действия (`accept`/`ready`) отбит **по состоянию**; прямой ingress на
>   `request_signature`/`push_root` отбит; неизвестный метод недопустим.
>
> **Что этим не закрыто и не может быть:** движение денег на Solana (это `two-outcome/tests/claim.rs`
> и живые `T5`/`F5`) и многопровайдерный RPC-консенсус (его нет вне IC mainnet, `07-build-plan §P7.5`).

> **Пересобран 2026-07-29 после `P7.6`** — прогоном, не чтением. §0 ниже несёт числа этого прогона;
> прежняя редакция открывалась списком «что ниже заведомо неверно» на десять пунктов, и по
> `01-standards §Документация` такой файл подлежал либо перепрогону, либо удалению. Перепрогнан.
>
> Что было исправлено по существу, а не пересчитано:
>
> - **§8.3 был неверен целиком.** Он утверждал, что `header_convention_holds` нет ни на одном шейпе.
>   Тест есть в **каждой живой форме** (`two-outcome`, `stream`) — юнит-тестом в `src/lib.rs`, то есть
>   без SBF-сборки, чтобы его нельзя было тихо пропустить в CI. Строка снята.
> - **Счётчики §1–§4 не пересчитывались** — таблицы сценариев остаются как есть, они про покрытие
>   спеки, а не про число тестов. Числа живут только в §0.
> - **Что этим проходом не прогонялось:** litesvm-наборы форм (`two-outcome`, `stream`) — им нужен
>   `cargo build-sbf`, и они не запускались; их строки в §0 помечены соответственно.

> **Поправка `P7.18` (аукцион: арифметический победитель).** Весь §1 ниже описывает аукцион до
> перехода на модель «побеждает самая дорогая ставка» и **как снимок остаётся верным** — но как
> описание текущего кода неверен целиком, а не по пунктам. Что перестало существовать: состояния
> `Performing`/`Voting`, методы `pick_winner`/`ready`/`vote`, `logic/verdict.rs`, `MIN_VOTE_WEIGHT`,
> `V_MAX`, `voting_period`/`perform_window`; значит, и все строки §1.1–§1.2, ссылающиеся на них,
> покрывают несуществующие сценарии. Что появилось: закрытие `Bidding → Done{winner_lot}` на `T` с
> `argmax total` по пригодным лотам, хранение `gross`/`total` (канистра **больше не слепа к суммам**),
> ничья в пользу открывшегося раньше. Действующий чеклист — `crown-games/auction/docs/spec.md §DoD`;
> прогон после перехода: logic 15 + канистра 30 + pocket-ic 10 = **55**, clippy strict/fmt/.did чисто.
> Не зачёркнуто построчно и не переписано по той же причине, по которой не переписан `P7.14`: молча
> отредактированный датированный снимок перестаёт быть свидетельством.

> **Поправка `P7.14`.** Пункты, зачёркнутые с меткой ᴾ⁷·¹⁴, описывают покрытие профиля получателя в
> `conditional-tasks`. Механизма больше нет (`07-build-plan.md ¹⁷`), поэтому нет и тестов: это **не
> пробел покрытия**, закрывать его нечем. Зачёркнуты, а не удалены, — файл датирован и читается как
> снимок прогона; молча отредактированный снимок перестаёт быть свидетельством. Числа §0 после этого
> прохода не пересчитывались.

**Что изменилось с прошлого прохода** (учтено ниже): (1) auction переработан на модель «получатель
называет победителя» (`pick_winner`) — финал-скан/`winner.rs`/`FinaleCursor`/`SCAN_CHUNK`/`FinaleDue`
удалены, spec.md приведена к коду → оба прежних расхождения §6 закрыты; (2) форма `stream` (ядро
subscription) получила litesvm-тесты ([lifecycle.rs](crown-factory/shapes/stream/solana/tests/lifecycle.rs),
6 шт., зелёные); (3) красный config-тест tasks починен (конфиг откатили к плейсхолдерам). **Бейзлайн
сейчас полностью зелёный.**

Метод: сверка **§Таблица переходов** и **§Крайние случаи** каждой спеки (полное пространство сценариев,
заданное самой спекой) против фактических `#[cfg(test)]`-модулей и `tests/`-директорий. Прогонял suite'ы
локально (`cargo test`, `POCKET_IC_BIN=~/.cache/dfinity/versions/0.32.0/pocket-ic`; форма `stream` —
`cargo test --test lifecycle` в `crown-factory/shapes/stream/solana`, требует `cargo build-sbf`).

## Легенда статусов

| Знак | Значение |
|---|---|
| ✅ | покрыто явным тестом (unit) |
| 🟡 | покрыто слабо/косвенно (напр. только через `done_is_absorbing` или соседнюю ветку `_ => Err`) |
| ❌ | не покрыто |
| 🔴 | тест есть, но **красный** (сломан) |
| 🔵 | требует слоя, которого нет: pocket-ic (эндпоинт) / devnet-e2e (деньги) / litesvm (ончейн-форма) |
| ⚠️ | спека↔код расхождение — **решение отдельно** (§Расхождения), тесты этих мест пока не трогаем |
| `[редк.]` | редкий / крайний сценарий |

---

## 0. Бейзлайн (прогон 2026-07-29, после `P7.6`)

| Реп / игра | logic-unit | canister-unit | pocket-ic | Σ прогнано |
|---|---|---|---|---|
| auction | 20 ✅ | 28 ✅ | 9 ✅ (`endpoint_e2e` 8 + `full_e2e` 1) | **57** |
| conditional-funding | 19 ✅ | 20 ✅ | 9 ✅ (`endpoint_e2e` 8 + `full_e2e` 1) | **48** |
| conditional-tasks | 19 ✅ | 25 ✅ | 12 ✅ (`birth_e2e` 3 + `endpoint_e2e` 7 + `full_e2e` 2) | **56** |
| `crown-games-common` | 38 ✅ | | | **38** |
| `crown-indexer` | 37 ✅ (unit) | | 8 ✅ (PocketIC) | **45** |
| `crown-reduce` | 8 ✅ (закон + property) | | | **8** |
| `crown-relay` | 9 ✅ (unit) | | | **9** |
| `crown-factory` (офчейн) | derive 2 · escrow 3 · salt 4+3 ✅ | | | **12** |

**Итого 273 зелёных, ноль `#[ignore]`.** Не прогонялось этим заходом: litesvm-наборы форм
(`two-outcome`, `stream`) и `crown-relay`/`crown-indexer` PocketIC-наборы сверх перечисленных —
первым нужен `cargo build-sbf`.

**Что изменили счётчики относительно прошлого снимка** (не регресс):

- `crown-games-common` 22 → **38**: в него переехали политика подписи (`signing`, 4 теста) и общие
  поля/`chain_id` (`field`, 2), плюс чтение конфига на бейке (`config_bake`, 4).
- `conditional-tasks` канистра 31 → **25**: удалён модуль-переходник `request.rs` (5 тестов, из
  которых 4 дублировали `common::request`; пятый — «реальное сообщение игры проходит общий
  wire-формат» — переехал в `protocol.rs`), и снята пер-игровая копия тестов хранилища подписи.
- `auction`/`conditional-funding` — та же копия тестов подписи снята; счётчики не изменились, потому
  что у них она и была меньше.
- `crown-indexer` 38 → **37**: удалена резервация ингеста (три её теста), добавлены два о том, что
  гарантию держит `applied`.

## 1. auction

Спека: [crown-games/auction/docs/spec.md](crown-games/auction/docs/spec.md).
Логика: [logic/src/](crown-games/auction/logic/src/) · Канистра: [canister/src/](crown-games/auction/canister/src/).
Модель «получатель выбирает победителя»: `Bidding→Performing→Voting→Done`, `pick_winner`, резолвер на
вкладе (`key([entry_id])`), канистра слепа к суммам. **Все тесты — in-src `#[cfg(test)]` (17 logic + 21
канистра); `tests/`-директории нет, pocket-ic/e2e нет.**

### 1.1 Таблица переходов (метод × состояние)

| Сценарий | Статус | Где / почему |
|---|---|---|
| `create_auction` — чистая деривация `auction_id` | 🟡 | байт-точность id (`protocol::auction_id_is_byte_exact_and_commits_every_field`); эндпоинт (`AuctionIdMismatch`, echo без записи) 🔵❌ |
| `get_resolver` (query, любое состояние) | 🟡 | деривация `entry_id` покрыта (`protocol::entry_id_is_a_unique_leaf_scope_per_entry`); сам query ❌ |
| `register_entry` Bidding OK (gross≥min_entry, дедлайн, лот не возвращён, пруф рождения) | ✅/🔵 | `state::add_entry_creates_lots_and_dedups_escrow`, `validate::gross_below_min_entry_is_rejected_first`, `validate::deadline_boundary...`; **пруф рождения** 🔵❌ |
| `register_entry` заморожен на `T` (state остаётся Bidding) | ✅ | `machine::registration_freezes_at_t_but_state_stays_bidding` |
| `register_entry` в Performing/Voting/Done → ✗ | ✅ | `machine::live_states_reject_registry_and_stray_actions` (Performing/Voting) + `done_is_absorbing` (Done) |
| долив в возвращённый лот / дубль-эскроу → ✗ | ✅ | `state::no_topup_into_a_returned_lot`, `state::add_entry_creates_lots_and_dedups_escrow` |
| `accept_lot` Bidding OK (не принят, не возвращён, recipient) | ✅ | `state::accept_gates_recipient_state_and_double_accept` |
| `pick_winner` из принятого лота (recipient) → Performing | ✅ | `state::pick_winner_opens_performing_and_gates`, `machine::pick_winner_opens_performing_before_or_after_t` |
| `pick_winner` возвращённого/непринятого лота → ✗ | ✅ | `state::a_returned_lot_cannot_be_picked` |
| `pick_winner` после отсечки `T` всё ещё работает | ✅ | `state::pick_after_bidding_close_still_works` |
| `return_lot` не-победитель, Bidding → returned | ✅ | `state::no_topup_into_a_returned_lot` (через возврат) |
| `return_lot` **победитель** в Performing → Done{Cancel} `[редк.]` | ✅ | machine + `state::winner_lot_return_and_entry_return_in_performing` (state-слой с `winner_lot`) |
| `return_entry` (один эскроу лота) Bidding OK | ✅ | `state::return_entry_marks_one_escrow` |
| `cancel_auction` Bidding → Done{null} | ✅ | `machine::cancel_from_bidding_is_no_winner`, `state::cancel_from_bidding_is_done_none` |
| `cancel_auction` прочие → ✗ | ✅ | `machine::live_states_reject_registry_and_stray_actions` (Performing/Voting) + Done |
| `ready` Performing → Voting{now} | ✅ | `machine::performing_times_out_to_cancel` (pre-timeout ветка) |
| `vote` Voting OK (порядок: sig→время→дедуп `(lot,voter)`→вес≥MIN) | ✅ | `machine::vote_only_in_voting_with_threshold_and_dedup` |
| `vote` прочие состояния → ✗ | ✅ | Bidding+Done + Performing (`machine::live_states...`, `state::ready_gates_bidding_and_add_vote_requires_voting`) |
| `request_signature` Bidding → «нет вердикта» без списания | 🟡 | `resolve`-ветка покрыта; эндпоинт 🔵❌ |
| `request_signature` Performing/Voting/Done → resolve+подпись `key([entry_id])` | 🟡 | `resolve` покрыт; путь подписи/оплаты 🔵❌ |

### 1.2 Вердикт two-outcome (`logic/verdict.rs`) — покрыт

| Сценарий | Статус |
|---|---|
| пусто/ничья → Cancel; строгое `Σdone>Σnot` → Settle | ✅ `empty_and_tie_are_cancel_strict_majority_settles` |
| переполнение суммы голосов → Cancel `[редк.]` | ✅ `overflow_is_cancel` |
| соответствие правилу (property-тест) | ✅ `matches_the_rule` |

### 1.3 Трёхступенчатое разрешение per-entry (`resolve`)

| Сценарий | Статус |
|---|---|
| ступень 1: возвращённый вклад → Cancel | ✅ `resolve_stage_one_and_two_are_cancel` |
| ступень 2: возвращённый лот → Cancel | ✅ там же |
| ступень 3: по состоянию × `is_winner_lot` (Done{Settle}: победитель→Settle, прочие→Cancel; Performing/Voting/Bidding → NoVerdict) | ✅ `resolve_stage_three_by_state_and_winner` |
| `entry_scope` per-вклад (лист), None для незнакомых | ✅ `state::entry_scope_is_per_entry_and_absent_for_unknowns` |
| ретрай пере-подписывает тот же вердикт (идемпотентность) | 🔵❌ |

### 1.4 Выбор победителя (`pick_winner`) — заменил финал-скан

| Сценарий | Статус |
|---|---|
| победителя называет получатель; ончейн-скана «кто дороже» нет | ✅ (по построению: `winner.rs`/`SCAN_CHUNK`/`FinaleCursor` удалены — нет ограниченных итераций во всей системе) |
| `pick_winner` только из принятого не-возвращённого лота → Performing{lot} | ✅ `state::pick_winner_opens_performing_and_gates`, `state::a_returned_lot_cannot_be_picked` |
| приём заморожен на `T`, но выбор возможен и после | ✅ `machine::registration_freezes_at_t_but_state_stays_bidding`, `state::pick_after_bidding_close_still_works` |
| застрявший в Bidding (не выбрали) → денег не держит, `refund()` по дедлайну | 🟡 (по построению — state держится; Performing-таймаут покрыт `machine::performing_times_out_to_cancel`) |

### 1.5 §Крайние случаи спеки — не покрыты

| Сценарий | Статус |
|---|---|
| escrow, рождённый до `T`, получает исход лота по деривации `[редк.]` | ❌ |
| долив в возвращённый лот → ✗ | ✅ `state::no_topup_into_a_returned_lot` |
| нулевые/пустые ставки → Done{null}, всем Cancel | ✅ `state::cancel_from_bidding_is_done_none` |
| «заявлено» (борд crown-app) никогда не влияет на исход | 🟡 (верно по построению — канистра слепа к суммам; не проверено ассертом) |
| кэп `V_MAX` голосов → VoteCapReached `[редк.]` | ✅ `state::add_vote_caps_at_v_max_and_wires_the_threshold` |
| самоподписанные `ready`/`cancel`/`pick_winner` не материализуют (→ NotFound) `[редк.]` | ✅ `state::unmaterialized_self_signed_actions_are_not_found` |

### 1.6 Эндпоинт / деньги (оплата `request_signature` ✅ pocket-ic §7; пруфы/подпись/деньги — 🔵 форж/devnet)

`register_entry` пруф рождения (`birth::certified_root`/`birth_from_witness`, `FieldMismatch`, `BadBirthProof`) ·
`vote` пруф веса (`voter_weight`) · `inspect_message` допуск голоса (нулевой/подпороговый отбит бесплатно) ·
`request_signature` (`Underpaid`, `WrongTarget`, `NotDecided`-без-списания, `SignFailed`, оплата-до-подписи,
ретрай-иммутабельность) · `accept_lot`/`pick_winner`/`return_lot`/`return_entry`/`cancel_auction`/`ready`
авторизация подписью recipient · `create_auction`/`AuctionIdMismatch` · Settle (сплит комиссии, репутация
донору) · Cancel (100% донору) · `refund()` по дедлайну — **пруфы/подпись/деньги ❌** (только косвенно через
чистые хелперы). **Закрыто pocket-ic** (`endpoint_e2e.rs`, §7): `Underpaid`/`WrongTarget`/`NotFound` без
списания, `inspect_message` отбивает мусорный `vote`, queries.

### 1.7 Закрыто в этом проходе (Слой 1, +7 тестов)

- **logic/`machine.rs` (+3):** `live_states_reject_registry_and_stray_actions` (✗-ячейки матрицы для
  Performing/Voting: register/accept/cancel/pick/ready/vote/return_entry-не-winner — больше не держатся на
  одном `done_is_absorbing`); `voting_tally_overflow_is_reported` и `performing_advance_overflow_is_reported`
  (overflow-ветки `advance` в Voting/Performing → `Overflow`, state цел).
- **canister/`state.rs` (+4):** `unmaterialized_self_signed_actions_are_not_found` (самоподписанные
  ready/cancel/pick/accept/return/vote на несуществующий id → `NotFound`, ноль записи);
  `winner_lot_return_and_entry_return_in_performing` (`return_entry` winner-лота в Performing OK + не-winner
  → ✗ + `return_lot` winner → `Done{Cancel}` на state-слое); `ready_gates_bidding_and_add_vote_requires_voting`
  (`ready` из Bidding ✗, `vote` до `ready` ✗); `add_vote_caps_at_v_max_and_wires_the_threshold` (кэп `V_MAX` +
  проброс порога веса). fmt/clippy strict чисто.

**Остаётся в auction** (не unit-уровень): §1.5 escrow рождённый до `T` (деривация), §1.6 весь эндпоинт-слой
(пруфы, `request_signature`, `inspect_message`, деньги) — это Слой 2 (pocket-ic) и Слой 3 (devnet).

---

## 2. conditional-funding

Спека: [crown-games/conditional-funding/docs/spec.md](crown-games/conditional-funding/docs/spec.md).
**Нет `canister/tests/` вовсе → 0 pocket-ic, 0 e2e.** `request.rs` нет (wire-парс живёт в common).
Материально тоньше tasks.

### 2.1 Машина + вердикт (кворум/порог)

| Сценарий | Статус |
|---|---|
| Funding, now<end × `ready` → Voting{now} | ✅ `ready_opens_voting_within_the_window` |
| Funding × `recipient_cancel` → Decided{Refund} | ✅ `recipient_cancel_refunds_from_funding` |
| Funding, now≥end × `ready`: tick→Refund, потом InvalidTransition (строгое `>=`) `[редк.]` | ✅ `funding_border_is_strict` |
| Funding, now≥end × `recipient_cancel` `[редк.]` | ✅ `recipient_cancel_after_deadline_is_refund_then_rejected` |
| Funding × `vote` → InvalidTransition | ✅ `vote_only_in_voting_dedup_and_threshold` |
| Voting × `vote`: вес<MIN → WeightBelowThreshold | ✅ (logic + `state.rs::add_vote_records_dedups_and_caps`) |
| Voting × `vote`: дедуп `(scope,voter)` → DuplicateVoter | ✅ |
| Voting × `vote`: валидная запись | ✅ |
| Voting × `ready`/`recipient_cancel` → InvalidTransition `[редк.]` | ✅ `voting_rejects_ready_and_cancel` |
| Voting, now≥voting_end: tally→Decided, поздний голос не считается `[редк.]` | ✅ `voting_tally_finalizes_with_quorum` |
| Voting закрылось пустым → **Settle** (тишина=выплата получателю) `[редк.]` | ✅ `silence_settles` |
| Voting нижняя граница (`voting_end−1` голос принят) `[редк.]` | ✅ `a_vote_at_the_last_instant_is_accepted` |
| Decided поглощающее | ✅ `decided_is_absorbing` |
| overflow `created_at+duration` → Overflow, state цел `[редк.]` | ✅ `overflow_leaves_state_untouched` |
| overflow `started_at+voting_period` (Voting) → Overflow `[редк.]` | ✅ `voting_side_overflow_leaves_state_untouched` |
| вердикт: пусто → **settle** `[редк.]` | ✅ `empty_settles` |
| вердикт: всё-против при кворуме чуть-не-набранном (Q−1)→settle vs набранном (Q)→refund `[ред./граница]` | ✅ `undershooting_quorum_settles` |
| вердикт: ничья→settle, «против» +1→refund (нестрогое `≥`) `[редк.]` | ✅ `tie_settles_and_a_quorate_no_majority_refunds` |
| вердикт: повышенный `approval_threshold` (7000) `[редк.]` | ✅ `a_higher_threshold_needs_more` |
| вердикт: overflow суммы yes/no → refund `[редк.]` | ✅ `overflow_of_a_sum_is_refund` (yes) + `every_counting_overflow_refunds` (no) |
| вердикт: overflow **явки** (`yes+no`) → refund `[редк.]` | ✅ `every_counting_overflow_refunds` |
| вердикт: **`checked_mul`** overflow доли/барьера → refund `[редк.]` | ✅ `every_counting_overflow_refunds` (share + bar ветки) |
| вердикт-property (кворум + **нестрогий** порог, refund ⇔ доля ниже барьера) | ✅ proptest `matches_the_rule` (веса ограничены) |

### 2.2 Канистра (state/validate/protocol/config) — покрыто

`collection_id` байт-точность + все поля (без gross/deadline) ✅ · сообщения байт-точны ✅ · вердикт-сообщение
байт-точно ✅ · реальная Ed25519-подпись + тампер ✅ · идемпотентная материализация (дубль→AlreadyExists) ✅
`materialize_is_idempotent` · только получатель `ready`/`cancel`, чужой→NotRecipient ✅ · `ready`/`cancel` на
неоматериализованном → NotFound `[редк.]` ✅ · `add_vote` дедуп+порог+**V_MAX** ✅ (cap=1) · ленивый Tick
финализирует Voting ✅ · границы duration включительны ✅ · дедлайн-граница точна + TimeOverflow ✅ ·
slot→created_at линейно+checked ✅ · deploy-инвариант (threshold∈[5000,10000), quorum≥MIN, fee<10000) ✅.

### 2.3 Не покрыто (эндпоинт/пруф — 🔵, + логические дыры)

**Закрыто pocket-ic** (`endpoint_e2e.rs`, §7): `Underpaid`/`WrongTarget`/`NotDecided` без списания,
`inspect_message` отбивает мусорный `vote`, queries. Ниже — оставшееся (форж пруфов/devnet):


Ленивое создание: **`create` без пруфа рождения не допускается границей вовсе** (эхо `Derived`
снято на `P8`) `[головной инвариант]` ✅ (`full_e2e`: беспруфовый `create` отбит до исполнения) ·
пруф рождения (`CollectionIdMismatch`/`BadBirthProof`/`FieldMismatch`/`CreatedAtOverflow`) 🔵❌ ·
`vote` `inspect_message` (нулевой/подпороговый бесплатно отбит) ❌ · `request_signature`
(`Underpaid`/`NotDecided`-без-списания/оплата-до-подписи/атомарность 1 подписи на N эскроу/`SignFailed`) 🔵❌ ·
иммутабельность+кэш подписи при ретрае ❌ · `donor==recipient` / самоголосование не блокируется ❌ ·
`bootstrap` идемпотентность / `init` mainnet-trap ❌ · N:1 мульти-эскроу поведение (одна подпись на всех) 🔵❌.

### 2.4 Закрыто в этом проходе (Слой 1, +5 тестов)

- **logic/`verdict.rs` (+1):** `every_counting_overflow_refunds` — три оставшиеся `checked_*` refund-ветки
  (no-корзина add, явка `yes+no`, `share=yes·10000` mul, `bar=threshold·turnout` mul).
- **logic/`machine.rs` (+4):** `voting_rejects_ready_and_cancel` (после `ready` двери нет);
  `recipient_cancel_after_deadline_is_refund_then_rejected` (tick→Refund на дедлайне);
  `voting_side_overflow_leaves_state_untouched` (Voting-clock overflow); `a_vote_at_the_last_instant_is_accepted`
  (нижняя граница окна). fmt/clippy strict чисто.

**Остаётся в funding:** весь §2.3 (эндпоинт-слой: ленивая ноль-запись, пруф рождения, `inspect_message`,
`request_signature`, `bootstrap`/`init`, N:1 мульти-эскроу) — Слой 2 (pocket-ic) и Слой 3 (devnet).

---

## 3. conditional-tasks (эталон)

Спека: [crown-games/conditional-tasks/docs/spec.md](crown-games/conditional-tasks/docs/spec.md).
Единственная игра с pocket-ic ([canister/tests/birth_e2e.rs](crown-games/conditional-tasks/canister/tests/birth_e2e.rs), 3 теста) и `request.rs` (5 wire-тестов). **Самое полное покрытие.**

### 3.1 Машина + вердикт

| Сценарий | Статус |
|---|---|
| Created × Accept → Accepted (текст раскрыт); Accepted × Ready → Voting{now} | ✅ `created_accepts_and_reveals_then_ready_opens_voting` |
| Created × Decline → Cancel; Accepted × Decline → Cancel `[редк.]` | ✅ `decline_cancels_from_created_and_from_accepted` |
| Accepted × Accept (двойной) → InvalidTransition, остаётся Accepted `[редк.]` | ✅ `double_accept_is_invalid_and_keeps_accepted` |
| Created × {Ready,Vote}; Accepted × Vote; Voting × {Accept,Ready} → InvalidTransition | ✅ `invalid_transitions_are_rejected` |
| **Voting × Decline → InvalidTransition** `[редк.]` | ✅ `invalid_transitions_are_rejected` (расширен: Accept+Decline+Ready) |
| Decided поглощающее; Tick no-op | ✅ `decided_is_absorbing` |
| Accept граница на `cutoff` точна (`cutoff−1` ок, `cutoff`→Cancel) `[граница]` | ✅ `accept_boundary_at_cutoff_is_exact` |
| Провал действия ≡ Tick (накопленный Cancel персистит) `[редк.]` | ✅ `failed_action_is_a_tick` |
| Vote после окна: tally→Settle, поздний голос отбит `[редк.]` | ✅ `vote_after_window_tallies_first_then_rejects` |
| `voting_end−1` голос ещё принят `[граница]` | ✅ `voting_end_boundary_is_exact` |
| Vote дедуп + вес<MIN → отбит, счёт цел `[редк.]` | ✅ `votes_dedup_and_threshold` |
| Clock overflow (deadline i64::MIN, cutoff underflow) → Overflow, state цел `[редк.]` | ✅ `time_overflow_leaves_state_untouched` |
| **`voting_end` overflow** (Voting) → Overflow `[редк.]` | ✅ `voting_end_overflow_leaves_state_untouched` |
| вердикт: пусто→**Settle**; ничья→**Settle**; not_done строго `>` →Cancel (501 vs 500) `[редк.]` | ✅ `empty_settles`, `tie_settles`, `a_strict_not_done_majority_cancels` |
| вердикт: только not_done → Cancel; только done → Settle | ✅ `only_not_done_is_cancel`, `only_done_settles` |
| вердикт: overflow done-суммы / not_done-суммы → Cancel (отказ подсчёта, не вердикт) `[редк.]` | ✅ обе ветки (`overflow_of_the_done_sum...`, `..._not_done_sum...`) |
| вердикт-property (cancel ⇔ not_done строго больше) | ✅ proptest `cancel_iff_not_done_strictly_greater` |

### 3.2 Канистра — покрыто

порядок отказов регистрации (~~Disabled→~~Floor~~→ProfileMin→Reputation~~→Duration→Deadline) ✅ ᴾ⁷·¹⁴ ·
~~репутация гейтится только при `min_reputation>0`, отсутствие пруфа = ниже, на-барьере проходит~~ ᴾ⁷·¹⁴ ·
границы duration
включительны + дедлайн-граница + TimeOverflow ✅ · идемпотентная материализация + text_hash ✅ · только
получатель, чужой→NotFound ✅ · verdict() None до Decided ✅ · ленивый Tick финализирует ✅ ·
~~счётчик профиля строго растёт, равный/ниже→StaleCounter~~ ᴾ⁷·¹⁴ · ~~профиль-кап, но существующие получатели
ещё обновляются~~ ᴾ⁷·¹⁴ · `add_vote` дедуп+порог+**V_MAX** ✅ · `task_id`
байт-точность+все поля (вкл. gross/deadline, 1:1) ✅ · length-prefix против неоднозначности canister/donor
`[редк.]` ✅ · сообщения байт-точны + различимы ✅ · реальная подпись+тампер ✅ · wire-парс (5 тестов:
signed↔extras, тампер, чужой подписант, отсутствие сепаратора) ✅ · game floor ≥ index MIN_GROSS ✅.

### 3.3 pocket-ic (`birth_e2e.rs`) — только первый хоп пруфа

✅ реальный сертификат индекса аутентифицирует combined root (BLS delegation→subnet→state-root vs NNS root) ·
✅ неверный root key отбит · ✅ неверный canister-id → нет certified data. **Не трогает** `register_task`,
деривацию адреса эскроу, `birth_from_witness`, `reputation_from_witness`, машину, `request_signature`.

### 3.4 §Крайние случаи спеки — не покрыты

| Сценарий | Статус |
|---|---|
| **иммутабельность вердикта**: исход пишется до первой подписи, ретрай подписывает тот же (Крайние случаи #9) | 🔵❌ (`request_signature` не тестируется) |
| **`donor == recipient` не блокируется** (Крайние случаи #10) `[редк.]` | ❌ |

(Прежний красный config-тест починен: конфиг [testnet.toml](crown-games/conditional-tasks/config/testnet.toml)
откатили к плейсхолдерам `aaaaa-aa`/`PLACEHOLDER_*` — боевые значения на P8/T5-прогоне; тест зелёный.)

### 3.5 Эндпоинт — 🔵 (нужен pocket-ic)

`register_task` (`TaskIdMismatch`/`BadBirthProof`/`FieldMismatch`/ленивая ноль-запись без пруфа) ·
~~`set_profile` (`ProfileMinBelowFloor`/`NotRecipient`/`StaleCounter`-обвязка)~~ ᴾ⁷·¹⁴ · `vote` `inspect_message` ·
`request_signature` (`NotDecided`-без-списания/`Underpaid`/оплата-до-подписи/ретрай-иммутабельность) ·
пруф репутации/веса (happy+fail) · `bootstrap`/`init` — **пруфы/деньги ❌**. **Закрыто pocket-ic**
(`endpoint_e2e.rs`, §7): `Underpaid`/`WrongTarget`/`NotDecided` без списания, `inspect_message` отбой, queries.

### 3.6 Закрыто в этом проходе (Слой 1, +1 тест)

- **logic/`machine.rs`:** `invalid_transitions_are_rejected` расширен на `Voting × Decline`;
  добавлен `voting_end_overflow_leaves_state_untouched` (Voting-clock overflow). fmt/clippy strict чисто.
- `donor == recipient` (§3.4) — не unit-тестируемо (это **отсутствие** проверки); достанется Слою 3 (devnet).

### 3.7 Комиссия — прейскурант игры (харнесс §9)

`fee_bps`/`fee_wallet` брались **из запроса**, а не из конфига — единственная из трёх игр (`auction` и
`conditional-funding` всегда подставляли `config::FEE_BPS`/`FEE_WALLET`). Пруф рождения это не ловил: он
свидетельствует лишь, что эскроу рождён с **той же** комиссией, что предъявил вызывающий. Исправлено —
обе константы идут из конфига и в `task_id`, и в соль эскроу.

| Сценарий | Статус |
|---|---|
| регистрация с `fee_wallet = свой` / `fee_bps = 0` → `TaskIdMismatch` (или отбой на границе) | 🔴 `full_e2e::register_decline_and_sign_a_real_verdict` — тест добавлен, но прогон упирается в незелёный `push_root` выше по сценарию |
| `auction` / `conditional-funding` берут комиссию из конфига | ✅ по коду (`config::FEE_BPS`/`FEE_WALLET` в `escrow_address`) |

---

## 4. subscription

Спека: [crown-games/subscription/docs/spec.md](crown-games/subscription/docs/spec.md). **Особая модель: канистры,
резолвера, threshold-подписи и `logic/`-крейта нет.** Вся логика — ончейн-форма `stream` в crown-factory:
[shapes/stream/solana/src/lib.rs](crown-factory/shapes/stream/solana/src/lib.rs) (670 строк, инструкции
`create_escrow`/`release`/`cancel`/`refund`). Тесты игры = litesvm-тесты формы `stream` + devnet-e2e.

**Текущее покрытие:** деривация соли — [salt/src/stream.rs](crown-factory/salt/src/stream.rs), 4 теста
(`byte_layout_is_exact`, `recipients_and_shares_are_positional`, `k_bounds_the_hashed_entries`,
`every_scalar_field_moves_the_salt`) **+ litesvm lifecycle формы** —
[lifecycle.rs](crown-factory/shapes/stream/solana/tests/lifecycle.rs), 6 тестов (зелёные, требуют
`cargo build-sbf`): `create` наполняет vault на `chunk·n`; `refund` после margin возвращает весь остаток
и закрывает; сторонняя «пыль» в vault не блокирует resolution (sweep донору); `cancel` подписью донора
дренирует остаток; чужая подпись отбита; полное расписание `release` платит через сплиттер + комиссию,
последний кусок settle+close. **happy-пути покрыты; редкие реверты и мульти-получатель — нет.**

### 4.1 §Действия и §Крайние случаи — статус

| Сценарий | Статус |
|---|---|
| `create_escrow` наполняет vault на `chunk·n` | ✅ `create_funds_the_vault_with_chunk_times_n` |
| `create_escrow`: валидация строки (1≤K≤6, Σshares≤10000, ≥1 ненулевая, piece≥MIN_GROSS, уникальность, fee<10000, chunk>0, n≥1, period>0) — **реверты** | ❌ (create только на валидной строке) |
| `create_escrow`: `salt`-mismatch → реверт | ✅ `create_rejects_a_salt_mismatch` |
| `release(k)` permissionless по расписанию, через сплиттер (net) + комиссия (fee) | ✅ `release_pays_recipients_through_the_splitter_and_settles` (`t0` в прошлом → все куски созрели) |
| `release`: последний кусок → `settled`, закрытие ATA (рента → донору) | ✅ там же (vault закрыт) |
| `release`: точный баланс (1 получатель 100%: `net+fee==chunk`, пыль=0) | ✅ там же |
| `release`: **мульти-получатель** — распределение по долям, пыль → донору, `Σ net + Σ fee + пыль == chunk` `[редк.]` | ✅ `multi_recipient_release_distributes_by_share` (60/40) |
| `release` **не в свой черёд** (`k ≠ released`) → реверт `[редк.]` | ✅ `release_out_of_order_is_rejected` |
| `release` **до срока** (`now < due`) → реверт `[редк.]` | ✅ `release_before_due_is_rejected` (будущий `t0`) |
| `release` после `cancel`/`settled` → реверт `[редк.]` | ✅ `release_after_settled_is_rejected` |
| `cancel`: подпись донора vs поле `donor`, остаток → донору, закрытие | ✅ `donor_signed_cancel_returns_the_remainder_and_closes` |
| `cancel`: **чужая/подменённая** подпись → реверт, vault цел | ✅ `a_foreign_cancel_signature_is_rejected` |
| `cancel`: повтор безвреден (терминальность) `[редк.]` | ❌ |
| `refund()` без подписи после margin: весь остаток → донору, закрытие | ✅ `refund_after_margin_returns_the_whole_remainder_and_closes` |
| `refund()`: сторонняя пыль в vault не блокирует (уходит донору) `[редк.]` | ✅ `dust_in_the_vault_does_not_lock_refund` |
| `refund()` **до просрочки** (`StreamAlive`) → реверт `[редк.]` | ✅ `refund_before_the_overdue_margin_is_rejected` |
| `due(t0,period,k)` checked, overflow→ошибка не паника, монотонность `[редк.]` | ❌ |
| `k ≥ n_chunks` / поля несуществующего эскроу → реверт, ничего не тратится `[редк.]` | ❌ |
| `donor == recipient` допустимо; program-owned получатель → реверт `[редк.]` | ❌ |
| гонка `cancel↔release` созревшего куска `[редк.]` | ❌ |
| `release`: подставной «токен-аккаунт» получателя (нужные байты, чужой владелец-программа) отбит **формой**, не сплиттером | ✅ `a_recipient_account_not_owned_by_the_token_program_is_rejected` (сверяет код ошибки 6016 — без этого тест зелен и без проверки) |

### 4.2 Закрыто в этом проходе (Слой 1, +6 litesvm)

- `release_out_of_order_is_rejected` (WrongChunk) · `release_before_due_is_rejected` (NotYetDue, будущий `t0`) ·
  `release_after_settled_is_rejected` (AlreadySettled) · `refund_before_the_overdue_margin_is_rejected`
  (StreamAlive) · `create_rejects_a_salt_mismatch` (SaltMismatch) · `multi_recipient_release_distributes_by_share`
  (60/40, инвариант точного баланса). fmt/clippy (флаги CI) чисто; требуют `cargo build-sbf`.
- **+4 (второй заход):** `create_rejects_every_invalid_row` (BadShares Σ>10000/all-zero, PieceBelowMin,
  DuplicateRecipient, ZeroChunk, BadSchedule n/period, BadFee + valid-control) · `partial_release_then_cancel`
  (остаток `(n−released)·chunk` донору — покрывает и гонку `cancel↔release`) · `a_repeated_cancel_on_a_settled_escrow`
  (AlreadySettled) · `donor_equal_recipient_is_allowed`.
- **Остаётся (осознанно отложено):** точная граница `now==due` (забракетирована `release_before_due` снизу +
  happy-release сверху); `due()` overflow (арифметика формы); program-owned получатель (ончейн-рантайм-свойство,
  на devnet).

### 4.3 Devnet e2e (S1) — DONE вживую

stream-программа задеплоена на Solana devnet (`81HsKu3F…`); драйвер `crown-games/subscription/e2e`
(`RpcClient`, донор `crown-index-e2e-donor`, тест-USDC `4zMMC9sr…`) прогнал вживую все три пути:
**release** (полное расписание через сплиттер: получателю net, комиссия fee_wallet, последний кусок
settle+закрытие) · **cancel** (подпись донора → остаток донору) · **refund** (permissionless после margin →
остаток донору). Балансы сверены on-chain, все tx подтверждены. Это закрывает §S1 build-plan. Остаётся S2 —
прод/mainnet (реальный Circle USDC).

---

## 5. Кросс-режущие пробелы (общие для всех)

1. **`canister/src/lib.rs` — 0 тестов во всех трёх канистрах.** Весь слой эндпоинтов, `inspect_message`,
   `request_signature`, `bootstrap`, `init`-trap не проверен в репо. Крупнейшая структурная дыра. → нужен
   pocket-ic харнесс (расширить паттерн `birth_e2e.rs`, портировать на auction/funding, где `pocket-ic`
   объявлен в dev-deps, но **не используется**).
2. **Инварианты `request_signature`** (неоплаченный не делает работы; «нет вердикта» без списания; оплата
   до подписи; одна подпись атомарно на N эскроу; ретрай-иммутабельность) — не тестируются нигде.
3. **Все `e2e/` пусты** при том, что DoD каждой спеки заканчивается «E2e через реальную сеть». Реального
   движения денег (сплит комиссии, кредит репутации, `refund()`) нет ни у кого.
4. **Форма `stream`: happy-пути + отказные реверты + create-валидация покрыты** (`lifecycle.rs`, 16 тестов, §4.2).
   Остаток мелкий: полная валидация строки в `create`, точная граница `now==due`, `due()` overflow, гонка
   `cancel↔release`, program-owned получатель.
5. ~~Матрица переходов держится на `done_is_absorbing`~~ — **закрыто Слоем 1** для auction/funding/tasks:
   ✗-ячейки не-Done состояний теперь имеют явные тесты (§1.7, §2.4, §3.6).

**Рекомендация по инструментам** (когда дойдём до написания): `cargo llvm-cov` по `logic/`+`canister/` для
охоты за непокрытыми ветками (`Err`/`checked_*→None`/`_ => Err` — там сидят редкие пути); model-based
proptest на случайные последовательности действий (ловит редкие *порядки*: Done-поглощение, монотонность
seq, иммутабельность вердикта, отсутствие паник); для формы `stream` — property/differential на инвариант
точного баланса (`Σ net + Σ fee + пыль + возврат == gross`) по случайным строкам распределения.

---

## 6. Расхождения спека↔код прошлого прохода — РАЗРЕШЕНЫ переработкой аукциона

(Новые, открытые на 2026-07-29, — в §8.)

Оба расхождения были в auction и оба **закрыты** переработкой на модель «получатель выбирает победителя»
(`docs/spec.md`, `pick_winner`; резолвер на вкладе). По закону проекта док приведён к коду тем же заходом.

### 6.1 Скоуп резолвера: `entry_id` — РАЗРЕШЕНО

`spec.md` приведён к коду: `resolver = key([entry_id])`, `entry_id = sha256(lot_id ‖ donor ‖ u64le(nonce))` —
одна подпись **на вклад** (`return_entry` разводит исходы вкладов лота, резолвер живёт на листе). Совпадает с
`auction/CLAUDE.md` и §Идентификаторы/§Вердикт `spec.md`. Расхождения нет.

### 6.2 Курсор финала — РАЗРЕШЕНО (моот)

Финал-скана в новой модели нет вовсе: победителя называет получатель (`pick_winner`), канистра лоты не
сканирует — `FinaleCursor`/стабильная память отпали. Расхождения нет.

---

## 7. Статус слоёв и что осталось

**Слой 0 — бейзлайн зелёный.** DONE (config-тест tasks починен).

**Слой 1 — редкие unit/ончейн-ветки.** DONE по всем 4 играм (+19 тестов: auction §1.7, funding §2.4,
tasks §3.6, stream §4.2). Детерминированно, без инфраструктуры. Матрицы переходов, overflow-ветки, кэп
`V_MAX`, самоподписанные→NotFound, отказные реверты формы `stream`, мульти-получатель — закрыты.

**Слой 2 — эндпоинт-слой канистр (`canister/src/lib.rs`).** Достижимая часть — **DONE**:
- *Прокси-харнесс (`endpoint_e2e.rs` + `e2e/relay-proxy`), +5 pocket-ic тестов на каждую из трёх канистр:*
  `request_signature` — `Underpaid` (неоплаченный не делает работы), `WrongTarget`, `NotFound`/`NotDecided`
  без списания; `inspect_message` отбивает мусорный `vote`; queries. Прокси нужен, т.к. `request_signature`
  вызывается только inter-canister (его multi-arg форму `inspect_message` отбивает на прямом ingress), а
  ingress не несёт циклов; фикстура-реле прикрепляет циклы. Зелёные, clippy strict чисто.
- *Осталось — нужен форж криптопруфов против индекса (дорого, пересекается со Слоем 3):* `register_entry`
  пруф рождения (`birth_from_witness`/`FieldMismatch`/`BadBirthProof`), `vote` пруф веса, threshold-подпись
  `Signed`, иммутабельность вердикта при ретрае, `bootstrap`/`init`.

**Слой 3 — e2e через devnet.** Инфраструктура Solana-периметра **поднята**: сплиттер `DKs2C9dR…` +
two-outcome фабрика `BGVQrwSw…` задеплоены на devnet, донор `crown-index-e2e-donor` фондирован (SOL + тест-USDC
`4zMMC9sr…`), индекс — на локальном dfx. (Первая пара — `EiLH5hCH…`/`9Cjb4fcy…` — мертва вместе с
потерянным ключом authority и перевыпущена на этих адресах; таблица переезда — `crown-factory/docs/spec.md §Заморозка`.)
- **`S1` (subscription) — DONE вживую на devnet.** stream-программа задеплоена (`81HsKu3F…`); драйвер
  `crown-games/subscription/e2e` прогнал `create_escrow → release(k) по расписанию через сплиттер → cancel
  подписью донора → refund()` с реальным USDC, балансы сверены on-chain (subscription build-plan §S1²).
- **`A5`/`F5`/`T5` (канистерные игры) — DONE вживую на devnet (`P7.5`).** Все три драйвера прогнаны:
  реальные эскроу, реальный вес голоса из книги (куплен настоящим `splitter.donate`), настоящие
  threshold-подписи `key_1` и ончейн `claim` — `settle`/`cancel`/`refund`, с проверкой балансов и
  закрытия vault. Детали — в пер-игровых build-plan'ах (`T5⁶`/`A5⁶`/`F5⁶`). Ниже — история того, как
  подписной путь был разблокирован.

  Подписной путь РАЗБЛОКИРОВАН, доказан. Фундамент поднят
  (локальная dfx-реплика, деплой индекса+tasks). Прежний «блокер» оказался **несовпадением имени ключа**:
  конфиг задавал `threshold_key = "dfx_test_key"`, а окружение выдаёт ключ под именем **`key_1`**.
  **Закрыто на `P7.5`:** `config/testnet.toml` всех трёх игр несёт `key_1`, и заведённый под обход
  профиль `pocketic` удалён как побайтовый дубликат — вместе с изолировавшим этот блокер тестом
  `canister/tests/schnorr_probe.rs`, всё содержимое которого (`bootstrap` → `KeyBootstrapped` →
  `get_resolver`) строго входит в `full_e2e`, доходящий до настоящей подписи.

  > **Исправление (2026-07-29).** Прежняя редакция утверждала, что «dfx 0.32 schnorr не поддерживает
  > вовсе», ссылаясь на ответ «unsupported management canister method». Это **неверно**, и ошибка была
  > в способе проверки: методы управляющей канистры нельзя звать ingress'ом из CLI, и отказ означает
  > «так вызывать не положено», а не «возможности нет». Из канистры всё работает — проверено
  > `dfx deploy` + `dfx canister call conditional-tasks bootstrap` → `KeyBootstrapped`.
  >
  > Существо дела: **локальная сеть dfx 0.32 сама работает на PocketIC** (в каталоге сети —
  > `pocket-ic-pid`/`pocket-ic-port`; флаг `dfx start --pocketic` потому и «has no effect»). Threshold
  > Schnorr в локальном окружении поддержан начиная с dfx 0.22. Верно узкое утверждение — **`dfx_test_key`
  > в этом окружении не заводится**, — а не широкое «локально подписать нельзя». Практическое следствие:
  > живой прогон воспроизводится обычным `dfx deploy`/`dfx canister call`, программный PocketIC-харнесс
  > для этого не обязателен.
  - ✅ **Конфиг-баг для деплоя — исправлен (`P7.5`).** Имя ключа зависит от окружения: `key_1` —
    локально и на IC mainnet, `test_key_1` — на тестовом сабнете IC; `dfx_test_key` не заводится нигде
    из трёх. Все три канистры (`config/*.toml`) переведены на `key_1`.
  - **D6+D7 — DONE (PocketIC, `full_e2e.rs`, 2 теста зелёные).** Мок SOL_RPC-канистра
    (`conditional-tasks/e2e/mock-sol-rpc`, на пиннутом principal `SOL_RPC`) отдаёт синтетическую
    `create_escrow`-tx → `index.ingest` (через relay-proxy) **сертифицирует рождение**. Затем полный IC-поток:
    tasks(`key_1`, `InitArgs{index, root_key}`) → `bootstrap` → `get_resolver` → инъекция рождения →
    `register_task`(подпись донора + cert + witness) → **Materialized** → `decline`(подпись получателя) →
    `request_signature` → **`Signed{Cancel}` — реальная 64-байтная Ed25519 threshold-подпись** через `key_1`.
    Закрыто: **D6** (пруф рождения потреблён register'ом), **D7** (`Signed` через реальный threshold-ключ).
  - ⚠️ **Находка:** проверка пруфа рождения (BLS-верификация сертификата) **превышает бюджет инструкций
    `inspect_message` (200M)** — прямой ingress `register_task` в этом окружении отбивается
    `CanisterInstructionLimitExceeded`. В тесте register маршрутизирован через прокси (inter-canister обходит
    inspect). Для прода это вопрос: user-ingress register должен укладываться в 200M — либо BLS не в
    inspect, либо register релей-фронтед. Стоит проверить на реальной реплике.
  - **D8 (кросс-чейн связь) — DONE.** `Signed{Cancel}`-подпись из tasks-канистры **проходит
    `verify_strict`** как Ed25519 над `VERDICT_DOMAIN ‖ program_id(32) ‖ outcome` для резолвера эскроу —
    ровно то, что проверяет ed25519-программа Solana и `two-outcome::claim::assert_resolver_signed`
    (`full_e2e.rs`). Сообщение совпадает по построению (config-равенство `DOMAIN+FACTORY == VERDICT_DOMAIN+
    crate::ID`). Само движение денег на claim (settle через сплиттер + комиссия, cancel 100% донору, forged/
    foreign-reject) уже litesvm-тестировано в `two-outcome/tests/claim.rs`. Т.о. каждое звено — подпись,
    её on-chain-валидность, и логика claim — проверено; **живой devnet-claim добавил бы лишь один on-chain
    tx, объединяющий уже-доказанные половины** (опциональная демонстрация, не новое покрытие).
  - **Итог канистерных игр:** D1–D4/D10 (unit), D6 (пруф рождения→register→materialize, PocketIC), D7
    (`Signed` через реальный threshold `key_1`, PocketIC), D8 (подпись on-chain-валидна + claim-логика
    litesvm) — **закрыты**. Паттерн (мок SOL_RPC + прокси + `key_1`-профиль) переносится на funding/auction.

**Правка поломки сборки (в этом заходе).** Другая сессия осознанно изменила `crown-games-common::Birth` на
`{donor, slot}` (убрала `gross` — он коммитится солью адреса эскроу, в birth-лист не пишется; см. коммент
`common/src/birth.rs`), обновив common+индекс, но оставив **все 3 канистры не компилирующимися** (читали
`b.gross`) и глоссарий стухшим. Закрыто: убрал избыточную `b.gross`-проверку в
`{auction,conditional-funding,conditional-tasks}/canister/src/lib.rs` (gross уже гарантирован адресом,
проверка `b.donor` осталась), обновил глоссарий `crown-spec/CLAUDE.md`. Все 3 игры пересобраны свежими против
current common: зелёные (tasks 29+3+5+19, funding 18+5+19, auction 26+5+20), clippy strict чисто.

---

## 8. Открытые расхождения спека↔код (сверка 2026-07-29)

Все три — **не** ошибки доков: доки приведены к коду, но сам код разошёлся между играми или с
экономической моделью. Решение принимается отдельно; до тех пор строки живут здесь и в
`07-build-plan.md §P6.6`.

### 8.1 `push_root`: РАЗРЕШЕНО — выровнено во всех трёх играх (2026-07-29)

Закрыто переносом расщеплённого доверия в обе отстающие игры: `auction` и `conditional-funding`
получили платный `push_root(cert)` + кеш `ROOTS` (`ROOT_CACHE = 4`, `root_price` в `config/`),
`RootPushed` в результат и в `.did`. Их `admit_register_entry` / `admit_create_collection` больше не
зовут `birth::certified_root` — реконструируют свидетеля против кешированного корня (чистый обход
хеш-дерева, O(log n)). Это инвариант харнесса, а не деталь эталона.

**Заодно закрыта дыра, которую нёс сам эталон:** путь **голоса** во всех трёх играх звал
`certified_root` из `voter_weight`, а `voter_weight` вызывается из `admit_vote` → `inspect_message`.
То есть две BLS-пары на анонимной границе на каждый голос — ровно то, что спека `conditional-tasks`
запрещает («BLS сюда не идёт — она на оплаченном `push_root`»). Регистрацию в эталоне починили, путь
голоса пропустили. Теперь `voter_weight` тоже ходит в кеш, и `cert` из wire-формата голоса ушёл.

Закреплено машинно: `certified_root` встречается **ровно один раз** в каждой игре — в `push_root`
(гейт CI «BLS only on the paid push_root», матричный job корневого workflow репа).

**Дедуплицировано (тот же заход).** Перенос сделали копированием: политика кеша (глубина
`ROOT_CACHE`, LRU-вытеснение, «свидетель проходит против любого корня») лежала в трёх играх тремя
побайтово одинаковыми копиями — девять мест на три игры. Вынесено в `crown-games-common::roots`
(`remember` / `birth` / `reputation`, 6 своих тестов); игра держит только сам `ROOTS` как состояние
канистры и свой платный эндпоинт (типы результата разные). Это не косметика: разъехавшись, копии
развели бы стоимость границы и ширину окна старого корня по играм — а это одна величина.
Локальные `const ROOT_CACHE` удалены из всех трёх.

Доказано прогоном, а не рассуждением: `full_e2e` каждой из трёх игр делает регистрацию/создание
**прямым ingress** (то, что до этого отбивалось бы `CanisterInstructionLimitExceeded`) и доходит до
реальной threshold-подписи `key_1`, проверенной `verify_strict` против резолвера эскроу. Плюс
эндпоинт-тесты: неоплаченный `push_root` не делает работы (`Underpaid`), оплаченный с кривым
сертификатом отбивается (`BadBirthProof`, оплата не возвращается), а регистрация со свидетелем **без
кешированного корня** отбивается на границе.

### 8.2 `MIN_GROSS[game]`: РАЗРЕШЕНО — механизм есть во всех трёх (2026-07-29)

> **Числа ниже — на дату среза; на `P7.22` (2026-07-31) они пересчитаны.** Механизм не изменился, а
> его вход — да: измеренный тариф `getTransaction` оказался вдвое выше того `g`, по которому эти флоры
> считались, и все три ушли на `2_200_000` (`cost.md §5`). Срез не переписываю — он датирован и
> описывает, что было верно тогда; актуальные значения всегда в `config/mainnet.toml` игры.

- `conditional-funding`: `config.min_gross = 410_000` (`cost.md §5`, $0.41), проверяется **первым** в
  `validate_registration` (`GrossBelowFloor`, в `.did`) — раньше флора не было вовсе.
- `auction`: `config.min_entry = 1_860_000` стал **платформенным** флором. Прежде `config::MIN_ENTRY`
  был забейкан, но **не использовался ни разу** — мёртвая константа, а фактический флор целиком
  задавал создатель полем прообраза. Теперь эффективный флор `= max(min_entry, MIN_ENTRY)`
  (`validate::entry_floor`): своё поле может флор только поднять. Резолвер здесь на вкладе, подпись
  не амортизируется — отсюда эталонные $1.86.
- У всех трёх был тест конфига `MIN_GROSS[game] ≥` индексного `MIN_GROSS` ($0.20). **Снят на
  `P7.13`** — он сравнивал чужой флор с литералом-копией, лежащей в том же файле, и покраснеть при
  подъёме индексного флора не мог физически. Сверка перенесена в шаг cost-gate `P8`, где оба числа
  перемеряются разом (`07-build-plan.md`); правило проекта — проверка, не способная покраснеть,
  хуже отсутствующей.

**Не считать закрытым числом.** Две нестыковки самой модели вынесены в `cost.md §6` как вход
cost-gate на P8: (1) §5 перечисляет Tasks/Funding/Subscription и не знает аукциона — список писался
до его переработки на per-entry резолвер; (2) для сбора §5 требует считать флор по худшему случаю
(`N = 1`), но публикует амортизированные $0.41. Механизм важнее числа: игры не заморожены, число
меняется передеплоем, а вот отсутствующего гейта на боевой канистре потом не дорисуешь.

### 8.3 Конвенция заголовка — РАЗРЕШЕНО (пункт был неверен)

Утверждение «такого теста нет ни на одном шейпе» ложно: тест есть в **каждой живой форме**, юнитом в
`src/lib.rs` — а не litesvm, чтобы не зависеть от SBF-сборки и не быть тихо пропущенным. Третий шейп,
`payout-table`, вместе со своим тестом удалён на `P7.6` (форма без потребителя).

**Но проверял он не тот шов, и это исправлено на `P7.12`.** `header_convention_holds` сверял раскладку
сериализованного **аккаунта** (`donor` 8..40, `salt` 40..72) и в комментарии ссылался на
`crown-indexer::recognize::decode_birth`, который этих байт не читает: индекс берёт всё из
**инструкции** — `donor` слот 0, `escrow` слот 1, `salt` данные 8..40 — и передеривирует PDA, а
`getAccountInfo` в его репе не встречается ни разу. Тест заменён на
`the_birth_layout_the_index_reads_holds`, который сверяет именно эти три позиции
(`crown-factory/docs/spec.md §Конвенция заголовка`, п. 7).

### 8.4 Инфраструктурные, но блокирующие деплой

- ~~`threshold_key = "dfx_test_key"`~~ — **закрыто на `P7.5`**: в `config/testnet.toml` всех трёх игр
  теперь `key_1`, как и в `mainnet.toml`. Имя ключа задаёт окружение (`key_1` локально и на IC mainnet,
  `test_key_1` — на тестовом сабнете), подписывать локально **можно** (см. исправление в §7), а профиль
  `pocketic`, существовавший ровно под обход неверного имени, удалён.
- ~~`IC_MAINNET_ROOT_KEY` — пустой срез во всех трёх канистрах~~ — **закрыто на `P8`**: ключ вбит одной
  копией в `crown-games-common::birth` (`&[u8; 133]`), значение сверено побайтово между `/api/v2/status`
  боевого узла IC и константой `IC_ROOT_KEY` в `ic-agent` DFINITY. Три копии, которые `P7.6` п.6 числил
  вынесенными, вынесены здесь.

**Из обоих не осталось ни одного** — оба инфраструктурных блокера деплоя закрыты.
