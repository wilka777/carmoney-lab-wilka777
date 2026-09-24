# Карта кода: правило по пробегу

## Участвующие файлы

- `backend/config/rules.php` задаёт:
    - допустимый пробег для валидации: `vehicle.max_mileage_km = 500000`;
    - LTV-пороги: `ltv.approve_max = 60.0`, `ltv.review_max = 85.0`.
- `backend/src/Domain/AssessmentService.php` координирует оценку заявки.
- `backend/src/Domain/ApplicationValidator.php` нормализует и валидирует входные поля.
- `backend/src/Domain/LtvCalculator.php` рассчитывает LTV.
- `backend/src/Domain/DecisionEngine.php` возвращает `approve`, `review` или `reject`.
- `backend/src/Domain/VehicleAge.php` вычисляет возраст автомобиля.
- `backend/src/Domain/VinValidator.php` проверяет формат VIN.
- `backend/src/Domain/ValidationException.php` используется, если заявка не проходит валидацию.

## Порядок расчёта

Вызов начинается с `AssessmentService::assess(array $payload)` в `backend/src/Domain/AssessmentService.php:28`.

1. Вызывается `ApplicationValidator::validate($payload)` (`AssessmentService.php:30`).
2. Внутри `validate()`:
    - нормализуется VIN: `strtoupper(trim(...))`;
    - вызывается `VinValidator::isValid($vin)`;
    - вызывается `VehicleAge::inYears($year)`;
    - валидируется год выпуска: не ранее `vehicle.min_year`, не в будущем, возраст не больше `vehicle.max_age_years`;
    - валидируется пробег;
    - валидируются оценочная стоимость, запрошенная сумма и срок;
    - если есть ошибки, выбрасывается `ValidationException`. В этом случае LTV и решение **не рассчитываются**.
    - если ошибок нет, возвращается нормализованный массив, включая `mileage`, `market_value` и `requested_amount`.
3. `AssessmentService::assess()` вызывает `LtvCalculator::calculate($input['requested_amount'], $input['market_value'])` (`AssessmentService.php:32`).
4. `LtvCalculator::calculate()` вычисляет `round(requested_amount / market_value * 100, 2)`.
5. `AssessmentService::assess()` передаёт LTV в `DecisionEngine::decide($ltv)` (`AssessmentService.php:33`).
6. `DecisionEngine::decide()` возвращает решение:
    - `approve`, если `LTV < approve_max`, то есть фактически **строго меньше 60.0**;
    - `review`, если LTV не меньше 60.0 и `LTV <= review_max`, то есть от **60.0 до 85.0 включительно**;
    - `reject`, если `LTV > 85.0`.
7. Затем `AssessmentService::assess()` повторно вызывает `VehicleAge::inYears($input['year'])` для поля ответа `vehicle_age`.
8. `approved_limit` равен запрошенной сумме только при `approve`; при `review` и `reject` это `0`.

Комментарий в `rules.php:39` утверждает, что `approve` выдаётся при `LTV <= approve_max`, но фактический код в `DecisionEngine::decide()` использует `<`. Поэтому при LTV ровно `60.0` текущий результат: `review`.

## Правило «пробег больше 400 000 км → review»

Такое правило относится к принятию решения, а не к валидации: заявка с пробегом `400001` должна остаться корректной и получить `review`, а не завершиться `ValidationException`.

Место для него: `DecisionEngine::decide()` в `backend/src/Domain/DecisionEngine.php`.

Чтобы правило влияло на решение, потребуется:

1. Передавать пробег из `AssessmentService::assess()` вместе с LTV.
2. Изменить сигнатуру `DecisionEngine::decide()`, чтобы она принимала пробег.
3. В `DecisionEngine::decide()` поставить проверку пробега **до LTV-проверок**:
    - если `mileage > 400000`, вернуть `self::REVIEW`;
    - иначе применить существующие ветки LTV.

Такое расположение означает, что превышение порога пробега принудительно возвращает `review` даже если LTV по текущим правилам дал бы `approve` или `reject`.

## Данные для этого правила

Уже есть:

- входное поле `payload['mileage']`;
- нормализованное значение `$input['mileage']`, возвращаемое `ApplicationValidator::validate()` (`ApplicationValidator.php:78`);
- передача `$input` внутри `AssessmentService::assess()`;
- текущая конфигурация лимита допустимого пробега `vehicle.max_mileage_km`.

Не хватает:

- порога именно для решения `review` при пробеге больше `400000`: в `backend/config/rules.php` его **нет**;
- передачи `mileage` в `DecisionEngine::decide()`: сейчас функция получает только `float $ltv`;
- логики, связывающей пробег с решением: её **нет**.

Если следовать правилу о хранении бизнес-чисел в конфигурации, значение `400000` должно быть добавлено в `rules.php` отдельным параметром и передано в `DecisionEngine`. Сейчас такого параметра нет.

## Что уже проверяется про пробег

В `ApplicationValidator::validate()` (`ApplicationValidator.php:43-46`):

- пробег берётся из `payload['mileage']` и приводится к `int`;
- если поле отсутствует, используется `-1`;
- заявка не проходит валидацию, если пробег меньше `0`;
- заявка не проходит валидацию, если пробег больше `500000`, взятого из `rules.php`;
- при такой ошибке выбрасывается `ValidationException`, решения `approve`, `review` или `reject` нет.

Пробег сейчас:

- не участвует в расчёте LTV;
- не влияет на `approve` / `review` / `reject`;
- не влияет на `approved_limit`;
- не используется для расчёта возраста;
- не проверяется как отдельное условие для `review`.

`backend/src/Domain/DecisionEngine.php`, вставка должна быть в начале `decide()`, до проверок LTV:

```php
public function decide(float $ltv): string
{
    if ($ltv < $this->approveMax) {
        return self::APPROVE;
    }

    if ($ltv <= $this->reviewMax) {
```

Для правила по пробегу потребуется изменить сигнатуру на получение `$mileage` и вставить условие сразу после `{`, до `if ($ltv < ...)`.
