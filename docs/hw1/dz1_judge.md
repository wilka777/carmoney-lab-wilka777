# ДЗ.1: отчёт агента-судьи

Проверку выполнил отдельный агент-судья на другой модели. Он не изменял файлы.
Проверены спецификация, тесты и реализация ветки `hw1/dz1-wilka777`.

## Покрытие REQ

| Требование | Покрытие | Вывод |
| --- | --- | --- |
| REQ-MILEAGE-01 | `AssessmentServiceTest::testAppliesMileageRuleToPotentialApprove()` и `DecisionEngineTest::testAppliesMileageRuleWithoutDowngradingReject()` | Границы 399999, 400000 и 400001 покрыты; для 400001 проверены `review` и лимит 0. |
| REQ-MILEAGE-02 | `AssessmentServiceTest::testKeepsRejectForHighLtvAndMileageAboveThreshold()` и набор высокого LTV в `DecisionEngineTest` | `reject` сохраняет приоритет над правилом пробега. |
| REQ-MILEAGE-03 | `ApplicationValidatorTest::testRejectsMissingMileage()` и `ApplicationValidatorTest::testRejectsMileageAboveValidationMaximum()` | Пустой пробег и пробег выше 500000 остаются ошибками валидации. |
| REQ-MILEAGE-04 | `AssessmentServiceTest::testAppliesMileageRuleToPotentialApprove()` | Для 400001 проверен лимит 0; API-формат в реализации не менялся. |

## Лишние требования

Не обнаружены. Изменения ограничены порогом 400000, передачей валидированного
пробега в расчёт решения и тестами. Валидационный предел 500000, API, фронтенд,
БД и seed-данные не изменялись.

## Замечания судьи

1. Для AC-MILEAGE-07 нет отдельного feature-теста, который сравнивает точный
   набор полей HTTP-ответа для `400001`; unit-тест проверяет лимит 0. Формат
   фактически сохранён, но такой тест сделал бы контракт явным.
2. Для AC-MILEAGE-05 исключение проверяется напрямую у валидатора. Порядок
   вызовов в сервисе гарантирует, что решение после ошибки не рассчитывается,
   но отдельный regression-тест сервиса мог бы закрепить это напрямую.
3. Известное расхождение LTV на границе 60.0 осталось вне этой задачи, как
   зафиксировано в intent.

## Итог

`make test`: 37 тестов, 49 assertions — успешно. `make lint` — успешно.
`git diff --check main...HEAD` — без ошибок.
