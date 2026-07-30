# Каталог сценариев игр — полное пространство

Цель: **исчерпывающе перечислить все возможные сценарии** каждой игры, чтобы пространство было
закрыто и ничего не терялось. Это вход для тестирования: [games-coverage-audit.md](games-coverage-audit.md)
отвечает «покрыт ли сценарий и на каком слое», а этот файл — «какие сценарии вообще есть».

Выведено **механически из кода**, не по памяти: каждое плечо `match` в машине состояний = сценарий;
полная сетка `состояние × действие`; каждая временна́я граница (`advance`); каждая ветка `verdict`/`resolve`;
каждый гейт формы `stream`; каждый вариант `*Error`/`*Result`. Алфавиты сняты из:
`*/logic/src/machine.rs`, `verdict.rs`, `*/canister/src/{state,lib}.rs`, `crown-factory/shapes/stream/solana/src/lib.rs`.

---

## Метод: 10 ортогональных измерений сценарного пространства

Полное пространство сценариев игры = осмысленное произведение этих измерений. Перечисляя вдоль каждого,
не пропускаешь ветки.

| # | Измерение | Что перечисляет |
|---|---|---|
| **D1** | Переходы машины | каждая пара `(состояние, действие)` → новый стейт **или** ошибка |
| **D2** | Временны́е границы | каждый таймер (`advance`) в точках **до / ровно / после**; overflow арифметики |
| **D3** | Вердикт (tally) | каждый способ, которым голоса свёртываются в исход (пусто/ничья/большинство/кворум/overflow) |
| **D4** | Разрешение клейма | каждая ветка `resolve`/`request_signature` на **один** эскроу (settle/cancel/no-verdict/refund) |
| **D5** | Авторизация | подписант валиден/нет (получатель vs чужой; подпись донора vs чужая); действие на неоматериализованном |
| **D6** | Пруфы | пруф рождения / веса: валиден / невалиден / отсутствует |
| **D7** | Оплата | оплачен / недоплачен / «нет вердикта» (без списания) / не тот таргет |
| **D8** | Деньги | на каждый терминальный исход: кому/сколько (сплиттер/комиссия/репутация/возврат), точный баланс |
| **D9** | Кратность N:1 | много эскроу одной области: одна подпись, но **расхождение** исходов per-эскроу |
| **D10** | Границы/overflow | включительность границ, `checked_*`→ошибка-не-паника, кэпы (`V_MAX`, `K_MAX`) |

Легенда исходов ниже: **OK** = валидный переход; **✗<Error>** = отвергается с этой ошибкой;
`[редк.]` = редкий/крайний. Гейты в скобках — предусловия валидности (проверяются в порядке кода).

---

## 1. auction (форма two-outcome, область = вклад, N эскроу на лот)

**Состояния:** `Bidding` · `Performing` · `Voting{started_at}` · `Done{winner: Settle|Cancel|None}`.
**Действия:** `RegisterEntry` `AcceptLot` `ReturnLot{is_winner}` `ReturnEntry{is_winner_lot}` `PickWinner{lot}` `CancelAuction` `Ready` `Vote` `Tick`.
**Ошибки шага:** `InvalidTransition` `WeightBelowThreshold` `DuplicateVoter` `Overflow`.
**Ошибки состояния:** `AlreadyExists` `NotFound` `NotRecipient` `LotNotFound` `LotReturned` `LotAlreadyAccepted` `LotNotAccepted` `EntryNotFound` `EntryReturned` `DuplicateEscrow` `VoteCapReached`.
`T = created_at+duration` (отсечка приёма); `perform_end = T+perform_window`; `voting_end = started_at+voting_period`.

### D1 — Переходы (состояние × действие)

| Состояние | Действие | Исход |
|---|---|---|
| Bidding (now<T) | RegisterEntry | OK (гейты: `gross≥min_entry`, `deadline≥T+pw+vp+margin`, лот не возвращён, эскроу уникален, **пруф рождения**) · ✗GrossBelowMinEntry · ✗DeadlineTooTight · ✗LotReturned · ✗DuplicateEscrow · ✗BadBirthProof/FieldMismatch |
| Bidding (now≥T) | RegisterEntry | ✗InvalidTransition (реестр заморожен на T) |
| Bidding (now<T) | AcceptLot | OK (лот есть, не принят, не возвращён, recipient) · ✗LotNotFound · ✗LotAlreadyAccepted · ✗LotReturned · ✗NotRecipient |
| Bidding (now≥T) | AcceptLot | ✗InvalidTransition |
| Bidding (now<T) | ReturnLot{false} | OK (лот→returned; recipient) · ✗NotRecipient · ✗LotNotFound · ✗LotReturned |
| Bidding (now<T) | ReturnEntry | OK (эскроу→returned) · ✗EntryNotFound · ✗EntryReturned |
| Bidding (любое время) | PickWinner{lot} | OK → **Performing{winner_lot}** (лот принят ∧ не возвращён; recipient) · ✗LotNotAccepted · ✗LotReturned · ✗LotNotFound · ✗NotRecipient |
| Bidding (любое) | CancelAuction | OK → **Done{None}** (recipient) |
| Bidding | Ready · Vote | ✗InvalidTransition |
| Performing | Ready | OK → **Voting{now}** (recipient) |
| Performing | ReturnLot{true} | OK → **Done{Cancel}** (возврат победителя; recipient) `[редк.]` |
| Performing | ReturnEntry{is_winner_lot:true} | OK (вернуть один эскроу лота-победителя) `[редк.]` |
| Performing | ReturnEntry{is_winner_lot:false} | ✗InvalidTransition (лот-проигравший уже cancel) `[редк.]` |
| Performing | ReturnLot{false} · RegisterEntry · AcceptLot · PickWinner · CancelAuction · Vote | ✗InvalidTransition |
| Voting | Vote | OK (не дубль ∧ `weight≥MIN_VOTE_WEIGHT` ∧ `len<V_MAX`) · ✗DuplicateVoter · ✗WeightBelowThreshold · ✗VoteCapReached |
| Voting | всё кроме Vote/Tick | ✗InvalidTransition |
| Done{любой} | любое действие | ✗InvalidTransition (поглощающее); Tick → no-op |

### D2 — Временны́е границы

| Граница | до | ровно / после |
|---|---|---|
| `T` (приём) | RegisterEntry/AcceptLot/ReturnLot/ReturnEntry разрешены | заморожены (✗); PickWinner/CancelAuction **остаются** доступны |
| `perform_end` (Performing) | Ready ещё работает | → **Done{Cancel}** (таймаут), Ready ✗ |
| `voting_end` (Voting) | Vote принят (вкл. `voting_end−1`) | → **Done{verdict}**, поздний Vote ✗ (tally первым) |
| overflow `T`/`perform_end`/`voting_end` | — | ✗Overflow, стейт **не** двигается `[редк.]` |

### D3 — Вердикт two-outcome (`verdict(votes)` на Voting→Done)

| Входы голосов | Исход |
|---|---|
| пусто | Cancel |
| `Σdone == Σnot` (ничья) | Cancel (строго `>`) |
| `Σdone > Σnot` | **Settle** |
| `Σdone < Σnot` | Cancel |
| overflow `Σdone` или `Σnot` | Cancel `[редк.]` |

### D4 — Разрешение одного эскроу (`resolve` / `request_signature`)

| Вклад | Лот | Состояние × победитель | Исход |
|---|---|---|---|
| returned | — | — | **Cancel** (ступень 1) |
| live | returned | — | **Cancel** (ступень 2) |
| live | live | Done{Settle} ∧ is_winner | **Settle** |
| live | live | Done{Settle} ∧ ¬is_winner | Cancel |
| live | live | Done{Cancel} / Done{None} | Cancel |
| live | live | Performing/Voting ∧ is_winner | **NoVerdict** (без списания) |
| live | live | Performing/Voting ∧ ¬is_winner | Cancel |
| live | live | Bidding | NoVerdict |
| unknown | unknown | терминал/проигрыш | Cancel (никогда не победитель) |
| — | — | ретрай того же клейма | тот же вердикт (идемпотентно) `[редк.]` |

### D5–D7 — Авторизация / пруфы / оплата

- **D5:** `accept/pick/return/cancel/ready` — подписант ≡ recipient (иначе ✗NotRecipient). Vote — подпись кошелька-голосующего. Действие на **неоматериализованном** id → ✗NotFound (самоподписанные не материализуют).
- **D6:** `register_entry` пруф рождения — валиден→материализация/добавление · невалиден→✗BadBirthProof · поля не сходятся→✗FieldMismatch · **без пруфа→ноль записи**. Vote пруф веса — `≥MIN`→допущен, иначе отбит в `inspect_message` (не доходит до update).
- **D7:** `request_signature` — `<SIGN_PRICE`→✗Underpaid (ноль работы) · чужой chain→✗WrongTarget · Bidding/NoVerdict→NotDecided (без списания) · оплачен+вердикт→принять оплату **потом** подпись · неизвестный эскроу→EntryNotFound (refund ончейн).

### D8 — Деньги на терминальный исход

| Исход | Деньги | Комиссия | Репутация | Событие |
|---|---|---|---|---|
| **Settle** (победитель) | `gross−fee`→получатель | `fee_bps·gross`→fee_wallet | донору `= gross−fee` | `Settled(payer=escrow)` |
| **Cancel** | 100% донору | нет | нет | нет |
| **refund()** (по `deadline`, без подписи) | остаток донору | нет | нет | нет |

### D9 — N:1 (лот = N эскроу, резолвер per-вклад)

- Одна область (лот) — много эскроу; подпись — **на вклад** (`key([entry_id])`), поэтому исходы расходятся.
- Победитель, у которого **часть вкладов возвращена**: каждый возвращённый → Cancel (ступень 1), Settle достаётся только не-возвращённым (несозданному эскроу — по деривации). `[редк.]`
- Эскроу, рождённый **до T**, получает исход своего лота по деривации, даже если добавлен поздно. `[редк.]`
- «Заявлено» (борд crown-app) **никогда** не влияет на исход — канистра слепа к суммам.

### D10 — Границы/overflow/кэпы

Включительность границ (D2); `checked_*`→Overflow (не паника); кэп `V_MAX=500` голосов/область; дедуп `(lot,voter)`.

---

## 2. conditional-funding (форма two-outcome, область = сбор, N эскроу, **с кворумом**)

**Состояния:** `Funding` · `Voting{started_at}` · `Decided{Settle|Refund}`.
**Действия:** `Ready` `RecipientCancel` `Vote` `Tick`. `funding_end = created_at+duration`.

### D1 — Переходы

| Состояние | Действие | Исход |
|---|---|---|
| Funding (now<end) | Ready | OK → **Voting{now}** (recipient) |
| Funding | RecipientCancel | OK → **Decided{Refund}** (recipient) |
| Funding (now≥end) | Ready · RecipientCancel | (advance: → Decided{Refund}) затем ✗InvalidTransition `[редк.]` |
| Funding | Vote | ✗InvalidTransition |
| Voting | Vote | OK (`weight≥MIN` ∧ не дубль ∧ `<V_MAX`) · ✗WeightBelowThreshold · ✗DuplicateVoter · ✗VoteCapReached |
| Voting | Ready · RecipientCancel | ✗InvalidTransition (после ready двери нет) `[редк.]` |
| Decided | любое | ✗InvalidTransition; Tick → no-op |

### D2 — Границы

| Граница | до / ровно-после |
|---|---|
| `funding_end` | Ready ок / → **Decided{Refund}** (строго `>=`), Ready ✗ |
| `voting_end` | Vote принят (вкл. `−1`) / → **Decided{verdict}**, поздний Vote ✗ |
| overflow `funding_end` / `started_at+voting_period` | ✗Overflow, стейт цел `[редк.]` |

### D3 — Вердикт с кворумом (`verdict(votes, quorum_weight, approval_threshold)`)

| Входы | Исход |
|---|---|
| пусто / `turnout(=Σyes+Σno) < quorum_weight` | Refund (тишина/недобор кворума) |
| `turnout ≥ quorum` ∧ `Σyes·10000 > threshold·turnout` | **Settle** |
| `turnout ≥ quorum` ∧ `Σyes·10000 ≤ threshold·turnout` (вкл. ничью) | Refund |
| кворум чуть-не-набран (`Q−1`) vs ровно (`Q`) | Refund vs Settle `[граница]` |
| повышенный `approval_threshold` (напр. 7000) | нужно больше yes `[редк.]` |
| overflow: `Σyes`/`Σno` add, явка `yes+no`, `yes·10000` (share) mul, `threshold·turnout` (bar) mul | Refund (все ветки) `[редк.]` |

### D4 — Разрешение (`request_signature`)

| Состояние сбора | Исход одного эскроу |
|---|---|
| verdict None (не Decided / неизвестен) | NotDecided (без списания) |
| Decided{Settle} | Settle |
| Decided{Refund} | Refund |

### D5–D10

- **D5:** `ready`/`recipient_cancel` — recipient (иначе ✗NotRecipient); на неоматериализованном → ✗NotFound.
- **D6:** пруф рождения первого вклада → материализация `Funding` (иначе ✗CollectionIdMismatch/BadBirthProof/FieldMismatch/CreatedAtOverflow); **без пруфа — эхо `Derived`, ноль записи**. Vote — пруф веса в `inspect_message`.
- **D7:** `request_signature` — Underpaid / WrongTarget / NotDecided-без-списания / оплата-до-подписи.
- **D8:** Settle → **всем** эскроу через сплиттер получателю (минус комиссия), репутация каждому донору; Refund → **всем** возврат; `refund()` по `deadline` — ончейн-safety.
- **D9 (N:1):** одна подпись области (`key([collection_id])`) переиспользуется всеми эскроу — **все** живут одним исходом (в отличие от auction, где per-вклад); добавление эскроу пере-подписи не требует.
- **D10:** `V_MAX`, дедуп `(collection,voter)`, `donor==recipient` не блокируется, границы включительны.

---

## 3. conditional-tasks (форма two-outcome, область = задание, **1 эскроу, B=1**) — эталон

**Состояния:** `Created` · `Accepted` · `Voting{started_at}` · `Decided{Settle|Cancel}`.
**Действия:** `Accept` `Decline` `Ready` `Vote` `Tick`. `cutoff = deadline−voting_period−margin`; `voting_end = deadline−margin`.

### D1 — Переходы

| Состояние | Действие | Исход |
|---|---|---|
| Created | Accept | OK → **Accepted** (текст раскрыт) |
| Created | Decline | OK → **Decided{Cancel}** |
| Created | Ready · Vote | ✗InvalidTransition |
| Accepted | Ready | OK → **Voting{now}** |
| Accepted | Decline | OK → **Decided{Cancel}** |
| Accepted | Accept (двойной) | ✗InvalidTransition, остаётся Accepted `[редк.]` |
| Accepted | Vote | ✗InvalidTransition |
| Voting | Vote | OK (`weight≥MIN` ∧ не дубль) · ✗WeightBelowThreshold · ✗DuplicateVoter · ✗VoteCapReached |
| Voting | Accept · Decline · Ready | ✗InvalidTransition |
| Decided | любое | ✗InvalidTransition; Tick → no-op |

### D2 — Границы

| Граница | до / ровно-после |
|---|---|
| `cutoff` (Created/Accepted) | Accept/Ready ок (вкл. `cutoff−1`) / → **Decided{Cancel}**, действие ✗ (провал≡Tick) |
| `voting_end` (Voting) | Vote принят (вкл. `−1`) / → **Decided{verdict}**, поздний Vote ✗ |
| overflow `cutoff` / `voting_end` | ✗Overflow, стейт цел `[редк.]` |

### D3 — Вердикт (`verdict(votes)`)

| Входы | Исход |
|---|---|
| пусто / только not_done / ничья `Σdone==Σnot` | Cancel |
| `Σdone > Σnot` (строго, напр. 501 vs 500) | **Settle** |
| overflow `Σdone` / `Σnot` | Cancel `[редк.]` |

### D4–D10

- **D4:** `request_signature` — verdict None→NotDecided (без списания); Decided{Settle}→Settle; Decided{Cancel}→Cancel.
- **D5:** `accept/decline/ready` — recipient (✗NotRecipient); неоматериализован→✗NotFound. Ручек получателя (`set_profile`) нет: условия приёма — фильтр клиента, не состояние канистры (`P7.14`).
- **D6:** пруф рождения (сертификат→корень→свидетель; ✗BadBirthProof/FieldMismatch/TaskIdMismatch; без пруфа — ноль записи). Регистрация с гейтами по порядку: Floor→Duration→Deadline — все платформенные, предпочтений получателя среди них нет. Пруф в регистрации ровно один: репутация доказывается только на `vote`.
- **D7:** Underpaid / WrongTarget / NotDecided-без-списания / ретрай-иммутабельность (исход пишется до первой подписи).
- **D8:** Settle → получателю через сплиттер (минус комиссия), репутация донору; Cancel → 100% донору; `refund()` по `deadline`.
- **D9:** B=1 — одна область/один эскроу, подпись **не** амортизируется.
- **D10:** `V_MAX`, дедуп `(task,voter)`, `donor==recipient` не блокируется, границы включительны, length-prefix против неоднозначности `task_id`.

---

## 4. subscription — форма `stream` (без канистры/резолвера/голосов; расписание = авторизация)

**Инструкции:** `create_escrow` · `release(k)` · `cancel` · `refund()`. Состояние — на аккаунте эскроу
(`released:u16`, `settled:bool`). `due(k) = t0 + k·period`; `refund_bound = t0 + released·period + RELEASE_MARGIN`.
**Ошибки:** `BadRow` `ZeroChunk` `BadSchedule` `BadFee` `BadShares` `PieceBelowMin` `DuplicateRecipient` `SaltMismatch` `AlreadySettled` `WrongChunk` `NotYetDue` `StreamAlive` `NoEd25519` `BadSignature` `WrongDonor` `CancelMismatch` `WrongRecipient` `WrongFeeWallet` `WrongSplitter` `Overflow`.

### D1 — Действия × гейты (нет машины — гейты формы)

| Инструкция | OK при | Реверты |
|---|---|---|
| **create_escrow** | `1≤K≤6`, len(recipients)==len(shares)==K, `Σshares≤10000` ∧ ≥1 ненулевая, каждый ненулевой `piece≥MIN_GROSS`, уникальные получатели, `chunk>0`, `n_chunks≥1`, `period>0`, `fee_bps<10000`, `salt==hashv(поля)` → фонд `chunk·n_chunks` | ✗BadRow · ✗BadShares · ✗PieceBelowMin · ✗DuplicateRecipient · ✗ZeroChunk · ✗BadSchedule · ✗BadFee · ✗SaltMismatch |
| **release(k)** | `!settled` ∧ `k==released` ∧ `now≥due(k)` ∧ splitter/fee_wallet/donor/recipient-ATA сходятся → распределение; последний `k`→`settled`+закрытие vault | ✗AlreadySettled · ✗WrongChunk (`k≠released`) · ✗NotYetDue (`now<due`) · ✗WrongSplitter · ✗WrongFeeWallet · ✗WrongDonor · ✗WrongRecipient · ✗BadRow |
| **cancel** | `!settled` ∧ ed25519-инструкция донора над `CANCEL_DOMAIN‖program_id‖escrow‖0x01` → остаток донору+закрытие | ✗AlreadySettled · ✗NoEd25519 · ✗BadSignature · ✗WrongDonor · ✗CancelMismatch |
| **refund()** | `!settled` ∧ `now > refund_bound` (permissionless, без подписи) → остаток донору+закрытие | ✗AlreadySettled · ✗StreamAlive (`now≤bound`) |

### D2/D10 — Расписание и границы

- `release(k)` строго по порядку: `k` только `==released`, `released` только растёт → повтор невозможен, `k≥n_chunks` неисполним.
- Граница `now==due` **включительна** (созрел). `due()`/`refund_bound` — `checked`, overflow→✗Overflow (не паника).
- Гонка `cancel↔release` созревшего куска: созревший можно `release` (штатно) **или** `cancel` останавливает будущие и возвращает остаток.

### D3 — (нет голосования/вердикта — расписание детерминировано)

### D4 — Терминальность

`settled` после последнего `release`, либо после `cancel`/`refund`. Повтор любой инструкции на `settled` → ✗AlreadySettled (безвреден).

### D8 — Деньги (распределение куска, `K_MAX=6`)

| Действие | Деньги | Комиссия | Репутация | Событие |
|---|---|---|---|---|
| **release(k)** | `piece_j=chunk·share_j/10000`, `net_j=piece_j−fee_j` → получателю **через сплиттер**; пыль `chunk−Σpiece`→донору | `Σfee_j`→fee_wallet (одним переводом, мимо сплиттера) | **донору** (`+Σpiece`) | `Settled(payer=escrow)` |
| **cancel** | весь остаток `(n−released)·chunk`→донору | нет | нет | нет |
| **refund()** | весь остаток→донору | нет | нет | нет |

Инвариант точного баланса на каждый `release`: `Σnet + Σfee + пыль == chunk`; по всему потоку `Σ через сплиттер + Σ комиссий + Σ пыли + возврат == gross`.

### D5/D9 — Авторизация / кратность

- **create/cancel** — подпись донора (create — обычная; cancel — ed25519 против поля `donor`; подмена→другой `salt`→другой адрес→неисполнимо). **release/refund** — permissionless (расписание/просрочка = авторизация).
- Один донор — один эскроу (1:1 на область), но до `K_MAX=6` получателей в куске. Несколько эскроу донор отменяет каждый отдельно.
- Крайние: `donor==recipient` допустимо; program-owned получатель → `release` ревертит (выход через cancel/refund); частичный `release` → потом `cancel` (остаток `(n−released)·chunk`); сторонняя «пыль» в vault не блокирует resolution.

---

## Как этим пользоваться

1. Каждая строка выше — **сценарий**, который должен быть проверен на подходящем слое.
2. Дешёвые/детерминированные сценарии (D1–D4, D10) — **unit/litesvm**; пруфы/оплата (D6–D7) — **pocket-ic**;
   деньги/полные потоки/ветвление исходов (D8, кросс-чейн) — **devnet-e2e**.
3. Правило для e2e: покрыть **ветвление исходов** (settle **и** cancel/refund, кворум набран **и** нет,
   возврат вклада), а не один happy-путь — иначе «код работает» доказан лишь частично.
4. Статус покрытия каждого сценария — в [games-coverage-audit.md](games-coverage-audit.md).
