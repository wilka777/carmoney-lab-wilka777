готов
1) Учебный сервис предварительной оценки заявки на заём под ПТС: принимает заявку, считает LTV и возвращает `approve` / `review` / `reject`.
2) Makefile: `make help`, `make up`, `make down`, `make ps`, `make logs`, `make install`, `make test`, `make lint`, `make seed`; docker-compose.yml: сервисы `backend` и `db` (MySQL 8), backend запускается командой `php -S 0.0.0.0:8080 -t backend/public backend/public/router.php`.
3) `backend/src/Domain` (решение рассчитывает `DecisionEngine.php`).
модель: training-2026-09-gpt-5.6-terra
