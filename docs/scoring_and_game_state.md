# Scoring and Game State

## 1. Назначение документа

Документ задаёт детерминированные правила обработки пользовательских сообщений, изменения Game State, расчёта трёх независимых результатов, использования PAEI, восстановления контакта, подсказок, завершения сценария и аудита событий.

Документ является нормативной спецификацией MVP: если правило не описано здесь или в versioned-конфигурации сценария, оно не должно реализовываться скрытым образом в коде или промпте.

## 2. Границы и разделение ответственности

PAEI описывает соответствие сообщения коммуникационным предпочтениям NPC. PAEI **не равен** доверию, согласию, эмоциям, агрессии, BATNA, цели сделки или итоговому состоянию.

```text
User message
  → Input validation
  → Evaluator: observable features + scenario flags
  → PAEI Engine: fit and relevance
  → Action Classifier
  → Game Engine: scoring and terminal checks
  → Reaction Model
  → AI Opponent response
```

Только `Game Engine` может изменять числовые параметры состояния. `AI Opponent`, UI и Evaluator не изменяют Game State напрямую.

## 3. Термины

- **Ход (`turn`)** — одно принятое пользовательское сообщение; запросы подсказок ходом не считаются.
- **`contact`** — текущая возможность продолжать продуктивное взаимодействие.
- **`resistance`** — игровое препятствие достижению цели; не является прямой оценкой эмоций NPC.
- **`progress`** — накопленное продвижение по целям сценария.
- **`mandatory_goal`** — обязательная цель; без её выполнения успех невозможен.
- **`optional_goal`** — дополнительная цель; влияет на `goal_result`, но не является обязательной для успеха, если сценарий не указал иное.
- **`critical_error`** — заранее формализованное нарушение сценарного правила.
- **`hard_failure`** — условие немедленного поражения, заданное сценарием.

## 4. Game State

| Поле | Тип | Диапазон | Начальное значение | Правило |
|---|---|---:|---:|---|
| `contact` | integer | 0–100 | из difficulty config | clamp после каждого изменения |
| `resistance` | integer | 0–100 | из difficulty config | clamp после каждого изменения |
| `progress` | integer | 0–100 | 0 | начисляется только по правилам целей/действий |
| `critical_errors` | integer | 0–N | 0 | +1 за каждую зарегистрированную критическую ошибку |
| `hints_used` | integer | 0–N | 0 | +1 при принятой подсказке |
| `turn_number` | integer | 0–N | 0 | увеличивается только после принятого хода |
| `phase` | enum | config | `opening` | меняется по phase rules |
| `game_status` | enum | — | `active` | `active`, `contact_lost`, `success`, `failure`, `cancelled`, `timeout` |
| `recovery_attempts_used` | integer | 0–limit | 0 | увеличивается при начале попытки recovery |
| `completed_mandatory_goals` | set | — | ∅ | хранит уникальные ID |
| `completed_optional_goals` | set | — | ∅ | хранит уникальные ID |
| `negotiation_quality` | number | 0–100 | 50 | рассчитывается из quality ledger |
| `paei_result` | number | 0–100 | 50 | рассчитывается из истории fit |
| `goal_result` | number | 0–100 | 0 | рассчитывается по формуле ниже |
| `final_result` | number/null | 0–100 | null | вычисляется после terminal check |

### 4.1. Инварианты

1. Все числовые поля после применения delta находятся в допустимом диапазоне.
2. `turn_number` не уменьшается и не увеличивается при запросе подсказки.
3. Нельзя изменить терминальное состояние новым ходом.
4. Нельзя дважды начислить одну и ту же цель или одно и то же событие по одному `client_turn_id`.
5. `state_after = clamp(state_before + state_delta)` с учётом целевых и терминальных правил.
6. Все изменения имеют `event_id`, `rule_id` и версии конфигурации.

## 5. Конфигурация сложности

Сложность изменяет только конфигурационные параметры. Базовые delta действий едины для всех режимов; сила негативного действия задаётся параметром и применяется явно.

| Параметр | Easy | Normal | Hard |
|---|---:|---:|---:|
| `initial_contact` | 80 | 70 | 60 |
| `initial_resistance` | 20 | 40 | 60 |
| `contact_loss_threshold` | 20 | 20 | 20 |
| `goal_threshold` | 80 | 80 | 80 |
| `hint_limit` | 5 | 3 | 2 |
| `turn_limit` | 15 | 12 | 10 |
| `critical_error_limit` | 4 | 3 | 2 |
| `negative_action_strength` | 4 | 6 | 8 |
| `recovery_attempt_limit` | 1 | 1 | 1 |
| `recovery_min_weighted_fit` | -1 | 0 | 1 |
| `recovery_requires_positive_progress` | false | true | true |
| `hint_quality_penalty` | 1 | 3 | 4 |

**Запрещено:** использовать в переходах жёсткие значения `12`, `3`, `6` или иные значения вместо параметров конфигурации.

## 6. Базовая таблица scoring

| `action_type` | contact | resistance | progress | quality_delta | critical_errors |
|---|---:|---:|---:|---:|---:|
| `strong_positive` | +8 | −8 | +12 | +10 | 0 |
| `positive` | +5 | −5 | +8 | +6 | 0 |
| `neutral` | 0 | 0 | +2 | 0 | 0 |
| `negative` | −N | +N | 0 | −5 | 0 |
| `critical_error` | −15 | +12 | −5 | −15 | +1 |
| `recovery_action` | +15 | −10 | +5 | +5 | 0 |

Где `N = negative_action_strength` из difficulty config. Это устраняет расхождение между фиксированным `−6` и параметрами сложности.

`critical_error` может дополнительно содержать scenario-specific delta, но итоговая delta должна быть явно указана в конфигурации и журнале события.

## 7. Наблюдаемые признаки

Evaluator возвращает только признаки, которые можно обосновать текстом сообщения и контекстом сценария. Для каждого признака используются значения `true`, `false` или `unknown`; `unknown` не считается ни положительным, ни отрицательным сигналом.

Минимальная схема:

```yaml
conversation_progress: forward | neutral | backward
actionability: high | medium | low | unknown
solution_orientation: solution | problem_only | mixed | unknown
result_clarity: clear | partial | unclear | unknown
deadline_clarity: clear | partial | absent | unknown
ownership: explicit | implied | absent | unknown
process_clarity: clear | partial | absent | unknown
evidence_quality: strong | medium | weak | absent | unknown
risk_identification: explicit | partial | absent | unknown
alternative_present: true | false | unknown
future_value: explicit | implied | absent | unknown
interest_acknowledgement: explicit | implied | absent | unknown
common_ground: explicit | implied | absent | unknown
perspective_taking: explicit | implied | absent | unknown
respectfulness: respectful | neutral | disrespectful | unknown
buy_in_check: explicit | absent | unknown
critical_flags: []
```

## 8. PAEI

### 8.1. Профильные признаки

| Профиль | Фокус | Основные признаки |
|---|---|---|
| P | результат и действие | результат, срок, следующий шаг, ответственность, практичность |
| A | порядок и управляемость | структура, последовательность, доказательства, риски, критерии, фиксация |
| E | возможности и будущее | альтернативы, варианты, инновации, будущая ценность, допустимый риск |
| I | совместность | признание интересов, общая основа, уважение, перспектива другой стороны, buy-in |

### 8.2. Расчёт fit

Для каждого профиля:

```text
raw_fit_X = Σ rule_contribution_X
fit_X = clamp(raw_fit_X, -2, +2)
```

Правила должны иметь уникальные `rule_id`, вес и условие. Отсутствие признака (`unknown`/`absent`) не приравнивается к отрицательному признаку. Противоречащие признаки суммируются до применения clamp.

Если сообщение соответствует нескольким профилям, сохраняются **все четыре** fit; для классификации используется агрегированный показатель, а не выбор одного «главного» профиля.

### 8.3. Веса

```yaml
profile_weights:
  P: 0.25
  A: 0.25
  E: 0.25
  I: 0.25
```

Обязательные проверки:

```text
weight_X >= 0
Σ weight_X = 1.0 ± 0.000001
```

### 8.4. Взвешенный fit

```text
weighted_paei_fit = Σ(weight_X * fit_X)
weighted_paei_fit = clamp(weighted_paei_fit, -2, +2)
```

Если fit по профилю отсутствует или невалиден, ход отклоняется как `evaluator_invalid`, а Game State не изменяется. Молчаливое подставление нуля запрещено.

### 8.5. PAEI result

Для каждого хода сохраняется `fit_X`. Среднее считается по принятым ходам:

```text
average_fit_X = mean(fit_X over accepted turns)
profile_result_X = clamp(50 + 25 * average_fit_X, 0, 100)
paei_result = round(Σ(weight_X * profile_result_X), 2)
```

`paei_result` не меняет `contact`, `resistance` или `progress` напрямую. Его влияние на классификацию происходит только через явно описанное правило `weighted_paei_fit`.

## 9. Классификация действий

Каждый принятый ход получает ровно один `action_type`.

Порядок:

1. Проверка валидности Evaluator.
2. Проверка scenario critical flags и hard constraints.
3. Если `game_status == contact_lost`, проверка recovery.
4. Определение направления `forward/neutral/backward`.
5. Расчёт `weighted_paei_fit`.
6. Применение таблицы классификации.

| Условие | Тип |
|---|---|
| есть подтверждённое критическое нарушение | `critical_error` |
| `contact_lost` и выполнены recovery conditions | `recovery_action` |
| forward + high actionability + solution + fit ≥ 1 | `strong_positive` |
| forward + fit ≥ 0 | `positive` |
| neutral и нет negative flag | `neutral` |
| backward или fit ≤ −1 | `negative` |
| иное валидное сообщение | `neutral` |

Последняя строка устраняет неявный `None`: каждое валидное сообщение получает класс.

## 10. Цели и progress

### 10.1. Правила целей

Каждая цель имеет:

```yaml
id: agree_on_realistic_deadline
type: mandatory | optional
completion_rule: explicit_scenario_predicate
progress_value: 0..100
max_once: true
```

`progress` не должен начисляться только за наличие общих положительных слов. Начисление выполняется при событии `goal_completed`, подтверждённом scenario predicate.

Для MVP рекомендуется:

- обязательные цели имеют `progress_value`, суммарно дающий 80–100;
- optional goals имеют отдельный бонус до `max_optional_goal_bonus`;
- повторное выполнение одной цели не начисляет прогресс повторно;
- частичное выполнение фиксируется как `in_progress`, но не считается completed.

### 10.2. Расчёт goal_result

```text
mandatory_ratio = completed_mandatory_weight / total_mandatory_weight
optional_bonus = min(
    completed_optional_bonus,
    max_optional_goal_bonus
)

base_goal_result = 100 * mandatory_ratio

goal_result = clamp(
    0.80 * base_goal_result
    + 0.20 * progress
    + optional_bonus
    - critical_error_penalty,
    0,
    100
)
```

Где:

```text
critical_error_penalty = critical_errors * 10
```

Успех определяется не `goal_result`, а отдельными `success_conditions` сценария.

## 11. Negotiation quality

Ведётся журнал `quality_ledger`. Каждый ход добавляет ровно одну quality delta из таблицы scoring. Подсказки применяют конфигурационный штраф.

```text
negotiation_quality = clamp(
    50
    + Σ(action_quality_delta)
    - hints_used * hint_quality_penalty,
    0,
    100
)
```

Подсказка не меняет `goal_result` и `paei_result` напрямую. В `training` штраф может отображаться отдельно и не использоваться для межсценарного сравнения.

## 12. Критические ошибки

Каждая ошибка имеет `error_code`, `rule_id`, доказательство, severity, delta и признак terminal.

| Код | Критерий |
|---|---|
| `HARD_CONSTRAINT_BREACH` | нарушено обязательное ограничение сценария |
| `FALSE_FACT_ASSERTION` | заявлен сценарно опровергаемый факт как достоверный |
| `UNAUTHORIZED_COMMITMENT` | обещано действие без полномочий |
| `PERSONAL_ATTACK` | оскорбление или личное унижение |
| `THREAT_OR_COERCION` | угроза, принуждение или недопустимое давление |
| `CONFIDENTIALITY_BREACH` | раскрыта закрытая информация |
| `MANDATORY_STEP_SKIPPED` | пропущен обязательный шаг при выполненных условиях его обязательности |

Один и тот же `error_code` по одному ходу регистрируется не более одного раза, если конфигурация сценария явно не разрешает несколько независимых нарушений.

## 13. Потеря и восстановление контакта

После применения delta:

```text
if contact <= contact_loss_threshold:
    game_status = contact_lost
```

При `contact_lost`:

- обычные действия блокируются;
- разрешается ровно один следующий пользовательский ход;
- `recovery_attempts_used` увеличивается при принятии recovery-хода;
- `progress` сохраняется;
- при неудаче recovery → `failure`;
- повторная потеря контакта после использованной попытки → `failure`.

### 13.1. Условия recovery

```text
current_status == contact_lost
AND recovery_attempts_used < recovery_attempt_limit
AND action_type_candidate == recovery_action
AND conversation_progress == forward
AND actionability == high
AND weighted_paei_fit >= recovery_min_weighted_fit
AND (recovery_requires_positive_progress == false OR progress_delta_candidate > 0)
```

`action_type_candidate` для recovery определяется отдельным `recovery_rule_set`, а не обычной таблицей. Для Hard требуется `weighted_paei_fit >= 1` согласно конфигурации.

## 14. Подсказки

Типы: `direction`, `context`, `communication`, `mistake`, `strategy`.

Ограничения:

1. Не содержат готовой реплики.
2. Не могут автоматически выполнить действие.
3. Не гарантируют успех.
4. Не обходят hard constraints.
5. Не доступны в terminal state.
6. Не чаще одной подсказки за ход.
7. Не превышают `hint_limit`.

При успешном запросе создаётся `hint_used`, увеличивается `hints_used` и применяется `hint_quality_penalty` из difficulty config.

## 15. Terminal conditions

### 15.1. Success

```text
all(mandatory_goals_completed)
AND progress >= goal_threshold
AND contact > contact_loss_threshold
AND critical_errors < critical_error_limit
AND hard_failure_condition == false
```

### 15.2. Failure

```text
critical_errors >= critical_error_limit
OR hard_failure_condition == true
OR failed_recovery == true
OR scenario_failure_condition == true
```

### 15.3. Timeout

```text
turn_number >= turn_limit
AND success_condition == false
AND game_status == active
```

Для `contact_lost` сначала обрабатывается разрешённый recovery-ход. Если он не принят или неуспешен, фиксируется `failure`, а не `timeout`.

### 15.4. Приоритет

```text
1. cancel
2. critical_error_limit
3. hard_failure
4. contact_lost / failed_recovery
5. success
6. timeout
```

Если после хода одновременно достигнуты success и критический лимит, результат — `failure`. Если достигнут success на последнем допустимом ходу без более приоритетного failure, результат — `success`.

## 16. Final result

Три показателя рассчитываются независимо:

- `goal_result` — достижение целей;
- `negotiation_quality` — качество поведения;
- `paei_result` — соответствие коммуникационным предпочтениям NPC.

```text
final_result = round(
    0.50 * goal_result
    + 0.30 * negotiation_quality
    + 0.20 * paei_result,
    2
)
```

Проверка весов:

```text
0.50 + 0.30 + 0.20 = 1.0
```

`final_result` отображается для всех завершённых сценариев, но `success/failure/timeout/cancelled` является отдельным статусом и не выводится из одного лишь числового балла. Для `active` и `contact_lost` итоговый балл не считается финальным.

## 17. Игровой цикл и фазы

Фазы:

```text
briefing → opening → diagnosis → bargaining → closing → result → debrief
```

Каждая фаза имеет `entry_condition`, `required_objectives`, `allowed_actions`, `exit_condition` и `max_turns`.

Рекомендуемая схема Normal: 8–12 ходов. Переход между фазами выполняется только при выполнении `exit_condition`; при невыполнении фаза может продолжаться до `turn_limit`.

Пользователь видит цель и ограничения сценария, но не видит внутренние веса PAEI NPC. После завершения показываются: статус, три независимых результата, final result, сильные действия, ошибки, подсказки и рекомендации.

## 18. Event schema

Каждое событие содержит:

```json
{
  "event_id": "evt-000007",
  "event_type": "action_processed",
  "event_version": "1.0",
  "session_id": "sess-001",
  "client_turn_id": "turn-07",
  "scenario_id": "scenario_001",
  "scenario_version": "1.0",
  "scoring_version": "1.0",
  "turn_number": 7,
  "phase": "bargaining",
  "action_type": "positive",
  "rule_ids": ["ACTION_POSITIVE_01"],
  "profile_weights": {"P": 0.25, "A": 0.25, "E": 0.25, "I": 0.25},
  "profile_fit": {"P": 2, "A": 1, "E": 0, "I": 1},
  "weighted_paei_fit": 1.0,
  "features": {},
  "critical_flags": [],
  "state_before": {},
  "state_delta": {},
  "state_after": {},
  "terminal_check": {
    "cancel": false,
    "critical_error_limit": false,
    "hard_failure": false,
    "contact_loss": false,
    "success": false,
    "timeout": false,
    "result": "active"
  }
}
```

Поддерживаемые типы: `session_started`, `turn_submitted`, `evaluator_completed`, `action_classified`, `action_processed`, `goal_completed`, `critical_error`, `hint_used`, `contact_lost`, `contact_recovered`, `scenario_completed`, `scenario_failed`, `scenario_timeout`, `cancel_action`, `evaluator_invalid`.

## 19. Evaluator contract

Evaluator обязан вернуть:

```json
{
  "schema_version": "1.0",
  "evaluator_version": "1.0",
  "action_type_candidate": "positive",
  "profile_fit": {"P": 1, "A": 0, "E": 1, "I": 0},
  "weighted_paei_fit": 0.5,
  "features": {},
  "critical_flags": [],
  "goal_candidates": [],
  "confidence": 0.82,
  "evidence_spans": []
}
```

Валидация:

- все обязательные поля присутствуют;
- fit целые числа в `[-2, 2]`;
- веса валидны;
- `confidence` в `[0,1]`;
- критические флаги ссылаются на известные `error_code`;
- при ошибке схема возвращает `evaluator_invalid`, состояние не меняется;
- результат сохраняется и при replay не вычисляется повторно.

Одинаковый `client_turn_id` обрабатывается идемпотентно: повторный запрос возвращает ранее сохранённый результат.

## 20. Scenario configuration

Минимальная конфигурация:

```yaml
scenario_id: scenario_001
scenario_version: "1.0"
mode: scenario
profile_weights: {P: 0.25, A: 0.25, E: 0.25, I: 0.25}
mandatory_goals:
  - id: agree_on_realistic_deadline
    weight: 50
    completion_rule: explicit_realistic_deadline
  - id: define_next_step
    weight: 50
    completion_rule: owner_and_next_step_defined
optional_goals:
  - id: preserve_relationship
    bonus: 5
    completion_rule: relationship_preserved
  - id: propose_alternative
    bonus: 5
    completion_rule: alternative_proposed
hard_constraints:
  - no_false_commitments
  - no_personal_attacks
critical_error_limit_by_difficulty: {Easy: 4, Normal: 3, Hard: 2}
phase_rules: {}
success_conditions: {}
failure_conditions: {}
```

## 21. Technical integration

```text
Client UI
 → Session API
 → Scenario Manager
 → Evaluator
 → PAEI Engine
 → Action Classifier
 → Game Engine
 → Event Store
 → Reaction Model
 → AI Opponent
```

Хранилища MVP: `sessions`, `scenarios`, `scenario_versions`, `turns`, `game_events`, `goal_progress`, `quality_ledger`.

`game_events` — источник аудита и восстановления. Версионируются сценарий, scoring, PAEI rules и evaluator schema.

## 22. Unit-test matrix

| Группа | Обязательные тесты |
|---|---|
| Boundaries | clamp для всех параметров, отрицательные и сверхмаксимальные значения |
| Difficulty | Easy/Normal/Hard используют собственные `turn_limit`, `critical_error_limit`, `negative_action_strength` |
| Classification | каждое валидное сообщение получает ровно один action type; нет `None` |
| Critical errors | известный код, одно начисление на ход, terminal priority |
| Goals | частичное выполнение, повторное выполнение, mandatory/optional разделение |
| Recovery | один разрешённый ход, пороги по сложности, неудача → failure |
| Contact | переход при `<= threshold`, сохранение progress |
| Hints | лимит, один раз за ход, штраф по сложности, запрет в terminal state |
| PAEI | fit range, unknown ≠ negative, сумма весов, конфликтующие признаки |
| Results | независимый расчёт goal/quality/paei, final result не отменяет failure |
| Idempotency | повторный `client_turn_id` не меняет state повторно |
| Replay | сохранённый Evaluator даёт идентичный state_after |
| Events | state_before + delta = state_after, версии и rule_ids присутствуют |

## 23. Контрольная таблица решений

| Решение | Зафиксированное правило |
|---|---|
| Difficulty limits | всегда берутся из config |
| Negative action | `−negative_action_strength`, а не фиксированное значение |
| Turn limit | `turn_number >= config.turn_limit` |
| Critical limit | `critical_errors >= config.critical_error_limit` |
| Recovery | отдельные условия и difficulty threshold |
| Goals | формализованные predicates и уникальные IDs |
| PAEI | четыре fit, обязательные веса, неизвестные признаки не штрафуют |
| Evaluator | schema validation, evidence, confidence, replay |
| Final result | числовой балл отделён от terminal status |
| UI | показывает статус, три результата, ошибки, подсказки и дебриф |

## 24. История изменений

| Версия | Дата | Изменения |
|---|---|---|
| 0.5 | 2026-09-18 | исходный MVP-черновик |
| 1.0 | 2026-09-18 | устранены расхождения лимитов и delta, формализованы цели, recovery, PAEI, Evaluator, terminal priority, idempotency, события и тесты |
