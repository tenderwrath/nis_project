# Словарь полей hotels.csv (119 390 строк  отмен 37.04%)

Роль: target - целевая; leak - удалено из данных; disputed - в файлах, но в основной модели не использовать; ok - признак.

| Поле | Тип | Смысл | % пропусков | Уникальных | Роль |
|---|---|---|---|---|---|
| hotel | VARCHAR | Тип отеля: City Hotel / Resort Hotel | 0.00 | 2 | ok |
| is_canceled | BIGINT | Бронь отменена (1) или нет (0); No-Show тоже = 1 | 0.00 | 2 | target |
| lead_time | BIGINT | Дней от создания брони до заезда | 0.00 | 475 | ok |
| arrival_date_year | BIGINT | Год заезда | 0.00 | 3 | ok |
| arrival_date_month | VARCHAR | Месяц заезда, текстом (July) | 0.00 | 12 | ok |
| arrival_date_week_number | BIGINT | Номер недели заезда | 0.00 | 53 | ok |
| arrival_date_day_of_month | BIGINT | День месяца заезда | 0.00 | 31 | ok |
| stays_in_weekend_nights | BIGINT | Ночей на выходных (сб-вс) | 0.00 | 17 | ok |
| stays_in_week_nights | BIGINT | Ночей в будни | 0.00 | 36 | ok |
| adults | BIGINT | Число взрослых | 0.00 | 14 | ok |
| children | BIGINT | Число детей (4 пропуска -> 0) | 0.00 | 5 | ok |
| babies | BIGINT | Число младенцев | 0.00 | 5 | ok |
| meal | VARCHAR | Тип питания: BB, HB, FB, SC/Undefined | 0.00 | 5 | ok |
| country | VARCHAR | Страна гостя, ISO-код (пропуск = неизвестно) | 0.41 | 180 | ok |
| market_segment | VARCHAR | Сегмент рынка (Online TA, Groups, ...) | 0.00 | 8 | ok |
| distribution_channel | VARCHAR | Канал продаж (TA/TO, Direct, ...) | 0.00 | 5 | ok |
| is_repeated_guest | BIGINT | Повторный гость (1/0) | 0.00 | 2 | ok |
| previous_cancellations | BIGINT | Число прошлых отмен гостя | 0.00 | 15 | ok |
| previous_bookings_not_canceled | BIGINT | Число прошлых неотменённых броней | 0.00 | 74 | ok |
| reserved_room_type | VARCHAR | Тип номера при бронировании (код) | 0.00 | 10 | ok |
| assigned_room_type | VARCHAR | Тип номера, выданный при заезде | 0.00 | 12 | leak |
| booking_changes | BIGINT | Число изменений брони после создания | 0.00 | 21 | disputed |
| deposit_type | VARCHAR | Депозит: No Deposit / Non Refund / Refundable | 0.00 | 3 | ok |
| agent | BIGINT | ID турагента (пропуск = без агента) | 13.69 | 327 | ok |
| company | BIGINT | ID компании-плательщика (пропуск = нет) | 94.31 | 354 | ok |
| days_in_waiting_list | BIGINT | Дней в листе ожидания | 0.00 | 125 | ok |
| customer_type | VARCHAR | Тип клиента: Transient, Contract, Group, ... | 0.00 | 4 | ok |
| adr | DOUBLE | Средняя цена за ночь | 0.00 | 9186 | ok |
| required_car_parking_spaces | BIGINT | Парковочных мест запрошено | 0.00 | 5 | leak |
| total_of_special_requests | BIGINT | Число особых пожеланий | 0.00 | 6 | disputed |
| reservation_status | VARCHAR | Итоговый статус: Check-Out / Canceled / No-Show | 0.00 | 3 | leak |
| reservation_status_date | DATE | Дата установки итогового статуса | 0.00 | 958 | leak |

## Решения по данным (30-31.10)

- **Утечки удалены:** `reservation_status`, `reservation_status_date` (известны только после факта), а также `assigned_room_type` (выдаётся при заезде) и `required_car_parking_spaces` (у всех отменённых = 0).
- **Спорные поля** `booking_changes`, `total_of_special_requests` оставлены в файлах, но копятся после создания брони. В основной модели рекомендуется обучать без них, а версию с ними показать как сравнение. Быстрая проверка (градиентный бустинг, один прогон): ROC-AUC на тесте 0,864 без них и 0,878 с ними. Это ориентир, не итоговая метрика.
- **Разбиение по времени:** обучение - заезды до 31.12.2016, тест - заезды 2017 года (январь-август).
- **Строки не удаляются** в таблице `clean` (нужны для витрины занятости по ночам). Для модели (`train.csv`, `test.csv`) убраны брони с флагами: `adr` <= 0 (1 960, отмен 10,9%, треть из них сегмент Complementary), выброс `adr` > 1000 (1), без гостей (180), с нулём ночей (715). На ROC-AUC этот фильтр практически не влияет (0,864 против 0,863).
- **Пропуски:** `children` -> 0; `agent`, `company` -> 0 («нет»); `country` -> `UNK`.
- **Дубликаты:** 31 994 полностью совпадающих строк (27%). Идентификатора брони нет, поэтому настоящий дубль не отличить от одинаковых групповых броней. Не удалялись; фиксируем как ограничение данных. Утечки между обучением и тестом они не создают: дата заезда входит в строку, дубль всегда попадает в ту же часть разбиения.
- **Особенности данных:** `No-Show` входит в `is_canceled` = 1; доля отмен в тесте выше (38,7% против 36,2% в обучении); `deposit_type` = Non Refund даёт 99,4% отмен (артефакт данных); тест покрывает только январь-август 2017.
