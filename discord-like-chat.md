
# Highload проект :  Чат с разбивкой на Сервера/Гильдии 
  
---
# 1. Тема и целевая аудитория:

**Тип сервиса :** массовый B2C real-time чат для больших сообществ.  
**Ключевая модель:** server-based "Гильдия/Сервер -> Канал -> Роль/Разрешение  -> Событие". 
**Реальные аналоги:**  
- [Discord](https://discord.com/company).
- [Slack](https://slack.com/) / [Microsoft Teams](https://www.microsoft.com/en-us/microsoft-teams/group-chat-software) - похожая структура "рабочее пространство ->  каналы", но B2B.  
  
**Целевая аудитория для расчётов клона** (данные как у Discord):   
- **MAU(месячные активные пользователи)**: [200,000,000 / мес (глобально)](https://backlinko.com/discord-users).
- **DAU(дневные активные пользователи)**: [~26,000,000 / день (глобально)](https://www.demandsage.com/discord-statistics/).
- Регионы: Северная Америка + Европа.
# 2. Архитектурные приколы:

1)  Массовая рассылка многим в реальном времени (massive real-time fan-out) по WebSocket - одно сообщение  -> много онлайн-сессий. [Нагрузка растёт с размером гильдии](https://discord.com/blog/maxjourney-pushing-discords-limits-with-a-million-plus-online-users-in-a-single-server). 
2) Read-states (состояние сообщений прочитано/ не прочитано): [отдельные кеши и частые обновления](https://discord.com/blog/why-discord-is-switching-from-go-to-rust). 
3) Хранение истории сообщений в огромном масштабе (партиционирование). [177 нод в Кассандре, триллионы сообщений](https://discord.com/blog/how-discord-stores-trillions-of-messages).
4) ACL(контроль доступа): [роли/переопределения канала/порядок применения](https://support.discord.com/hc/en-us/articles/206141927-How-is-the-permission-hierarchy-structured).
  
# 3. MVP  - 7 основных функций:

1) Регистрация/логин + сессии (JWT(токены) / обновления))  .
2) Создание гильдии и вступление по invite-link.
3) Список серверов/каналов с учётом прав доступа.
4) История канала постранично(pagination)  .
5) Отправка сообщения + доставка в реальном времени (WebSocket)  
6) Прочитано/не прочитано (read-states). Упоминания пользователя в канале гильдии (теги через @).
7) Роли/права + модерация + поиск.
  
---  
# 4. Расчёт нагрузки:
## 4.1 Продуктовые метрики и допущения:

- **DAU = 10,000,000**  
- Открытия каналов (загрузка истории): `channels_open = 10 / user / day`  
- Запросы списков (guilds/channels): `channels_list = 5 / user / day`  
- [Доля пишущих сейчас (Discord benchmark of 30% communicators as a healthy goal)](https://discord.com/community/understanding-server-insights): `share_communicators = 0.30`, `DAU_communicators = 3,000,000`  
- Сообщений на пишущего (communicator): `messeges_per_communicator = 15 / day`  
  
### Продуктовые метрики:

| Метрика                   |          Значение |
| ------------------------- | ----------------: |
| MAU                       |  50,000,000 / мес |
| DAU                       | 10,000,000 / день |
| share_communicators       |              0.30 |
| DAU_communicators         |  3,000,000 / день |
| channels_open             |   10 / user / day |
| channels_list             |    5 / user / day |
| messeges_per_communicator |          15 / day |
  
## 4.2 RPS(запросов в секунду) по API:

Формулы:  
`request/day = DAU * actions_per_user_per_day`  
`RPS_average = request/day / 86400`  
`RPS_peak = RPS_average × 10` (пиковый коэффициент = 10)  
  
### RPS (average/peak):

| Операция                                           | request/day | RPS_average | RPS_peak |
| -------------------------------------------------- | ----------: | ----------: | -------: |
| GET /channels/{id}/messages (history)              | 100,000,000 |       1,157 |   11,570 |
| GET /me/guilds + GET /guilds/{id}/channels (lists) |  50,000,000 |         579 |    5,790 |
| POST /channels/{id}/messages (send)                |  45,000,000 |         521 |    5,210 |

## 4.3 Внутренний hot path (плотная нагрузка):

Допущение для рассылки (fan-out): `F = 20` доставок  на `1` сообщение.  

На **одно отправленное сообщение** извне - внутри системы превращается в пачку действий:
1. Записать в БД/лог (1 раз).
2. Отправить по WebSocket всем онлайн, кто видит канал (вот это и называется fan-out).
3. Обновить unread/mentions (read-states) для этих получателей (часто тоже “на каждого получателя”).

| Внутренние операции                                                |  events/day | RPS_average | RPS_peak |
| ------------------------------------------------------------------ | ----------: | ----------: | -------: |
| WebSocket deliveries = `messages/day * F`                          | 900,000,000 |      10,417 |  104,200 |
| Read-state (обновления счётчиков прочитки) = `~ messages/day *  F` | 900,000,000 |      10,417 |  104,200 |
|                                                                    |             |             |          |
  
> Примечание: `F` допущение, не взято из источника, это я написал на своих вайбах после использования сервиса. Так как судя по среднем каналам у одного сообщения есть минимум 20 получателей онлайн

---
# Источники:

1. https://discord.com/company
2. https://slack.com/
3. https://www.microsoft.com/en-us/microsoft-teams/group-chat-software
4. https://backlinko.com/discord-users
5. https://www.demandsage.com/discord-statistics/
6. https://discord.com/blog/maxjourney-pushing-discords-limits-with-a-million-plus-online-users-in-a-single-server
7. https://discord.com/blog/why-discord-is-switching-from-go-to-rust
8. https://discord.com/blog/how-discord-stores-trillions-of-messages
9. https://support.discord.com/hc/en-us/articles/206141927-How-is-the-permission-hierarchy-structured
10. https://discord.com/community/understanding-server-insights