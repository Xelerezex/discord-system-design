
---
## Покрытие типов данных

|Класс данных|Есть в MVP|Где хранится|
|---|--:|---|
|Постоянные пользовательские данные|Да|`users`|
|Данные авторизации и устройств|Да|`user_sessions`|
|Метаданные чатов|Да|`chats`|
|Состав участников чатов|Да|`chat_members`|
|История текстовых сообщений|Да|`messages`|
|Буферы доставки и синхронизации|Да|`update_buffer`|
|Кеш онлайн-статуса|Да|`presence_cache`|
|Журнал событий|Да|`event_log`|
|Файловые данные|Нет|В рамках данного MVP не используются и не проектируются|

## Полное описание таблиц

|Таблица|Назначение|Что хранится|Где именно хранятся данные|Особенности|
|---|---|---|---|---|
|`users`|Профиль пользователя|`id`, `username`, `phone_number`, `display_name`, `bio`, `created_at`, `updated_at`|Основное транзакционное хранилище|Постоянные данные учетной записи|
|`user_sessions`|Авторизация и активные устройства|`token`, `user_id`, `device_id`, `user_agent`, `ip_address`, `created_at`, `expires_at`|Быстрое session/key-value хранилище с TTL|Одна учетная запись может иметь несколько активных устройств|
|`chats`|Метаданные диалогов, групп и каналов|`id`, `type`, `title`, `owner_id`, `last_message_id`, `is_secret`, `created_at`, `updated_at`|Основное транзакционное хранилище|`type` различает private/group/channel; `is_secret` отмечает секретные чаты|
|`chat_members`|Участники чатов и их состояние|`chat_id`, `user_id`, `role`, `last_read_seq_no`, `unread_count`, `joined_at`, `updated_at`|Основное транзакционное хранилище|Хранит ACL и пользовательское состояние в чате|
|`messages`|История текстовых сообщений|`id`, `chat_id`, `sender_id`, `seq_no`, `body`, `is_edited`, `is_encrypted`, `created_at`, `updated_at`|Основное транзакционное хранилище|Для secret chats сервер хранит ciphertext, а `is_encrypted = true`|
|`update_buffer`|Буфер обновлений для доставки и синхронизации|`id`, `user_id`, `chat_id`, `message_id`, `update_type`, `payload`, `created_at`, `expires_at`|TTL-буфер|Используется для `getUpdates`/reconnect/sync|
|`presence_cache`|Онлайн-статус и last seen|`user_id`, `device_id`, `status`, `last_seen_at`, `expires_at`|In-memory cache с TTL|Нужен для realtime-логики|
|`event_log`|Журнал системных и пользовательских событий|`id`, `user_id`, `chat_id`, `message_id`, `event_type`, `payload`, `created_at`|Append-only событийное хранилище|Используется для аудита и аналитики|

- **plaintext** — исходное обычное сообщение, которое можно прочитать 
- **ciphertext** — результат шифрования, набор данных, который без ключа не читается
- **QPS** — это **Queries Per Second**, то есть **количество запросов в секунду**.