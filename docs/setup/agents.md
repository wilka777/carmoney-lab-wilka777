# Роли агентов: planner и scout

## Роли и права

- `planner` — основной агент для планирования. Может писать только в `docs/plan/`; правка любых других файлов и bash запрещены.
- `scout` — субагент-разведчик. Только читает и возвращает пути, строки и краткие описания; правка файлов и bash запрещены.

## Результат вызова scout: места с `mileage`

- `frontend/index.html:31` — поле формы `<input name="mileage">`.
- `frontend/app.js:8, 13–15` — пробег включён в числовые поля и попадает в JSON-запрос через `FormData`.
- `backend/src/Domain/ApplicationValidator.php:43–45` — чтение `payload['mileage']`, приведение к `int` и проверка диапазона `0…max_mileage_km`.
- `backend/src/Domain/ApplicationValidator.php:78` — пробег возвращается в нормализованном входном массиве.
- `backend/src/Domain/AssessmentService.php:30–33` — получает нормализованный `$input` с пробегом, но не передаёт его в логику решения.
- `backend/src/Repository/ApplicationRepository.php:45` — чтение `$input['mileage']` при сохранении в БД.
- `backend/src/Repository/ApplicationRepository.php:68` — выборка `v.mileage_km` при получении заявки.
- `db/schema.sql:22` — столбец `vehicles.mileage_km`.
- `db/seed.sql:31+` — синтетические значения пробега при заполнении таблицы.
- `backend/config/rules.php:23` — лимит `vehicle.max_mileage_km = 500000`, используемый валидатором.
- `tests/Unit/ApplicationValidatorTest.php:34`, `tests/Unit/AssessmentServiceTest.php:38` — тестовые входные значения пробега.

Пробег участвует в форме, валидации и сохранении/чтении БД, но сейчас не влияет на LTV или решение `approve` / `review` / `reject`.
