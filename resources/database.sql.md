```sql
Project telegram_clone_mvp {
	database_type: "PostgreSQL"
	Note: '''
		Логическая схема БД для высоконагруженного мессенджера-клона Telegram.
	'''

}
  
Table users {
	id uuid [pk, not null]
	username varchar(64) [not null, unique]
	phone_number varchar(32) [not null, unique]
	display_name varchar(128) [not null]
	avatar_file_id uuid
	bio varchar(512)
	created_at timestamp [not null]
	updated_at timestamp [not null]

	Note: 'Профиль пользователя'
	
	Indexes {
		username [name: 'idx_users_username', unique]
		phone_number [name: 'idx_users_phone_number', unique]
	}
}

  

Table user_sessions {
	token varchar(128) [pk, not null]
	user_id uuid [not null]
	device_id varchar(128) [not null]
	user_agent varchar(255) [not null]
	ip_address varchar(64) [not null]
	created_at timestamp [not null]
	expires_at timestamp [not null]
	
	Note: 'Активные пользовательские сессии и авторизация'
	
	Indexes {
		user_id [name: 'idx_user_sessions_user_id']
		expires_at [name: 'idx_user_sessions_expires_at']
	}
}

Table chats {
	id uuid [pk, not null]
	type varchar(32) [not null, note: 'private | group | channel']
	title varchar(255) [not null]
	owner_id uuid [not null]
	chat_photo_file_id uuid
	last_message_id uuid
	is_secret boolean [not null, default: false]
	created_at timestamp [not null]
	updated_at timestamp [not null]

	Note: 'Метаданные чатов, групп и каналов'
	
	  
	
	Indexes {
		owner_id [name: 'idx_chats_owner_id']
		updated_at [name: 'idx_chats_updated_at']
	}
}

Table chat_members {
	chat_id uuid [not null]
	user_id uuid [not null]
	role varchar(32) [not null, note: 'owner | admin | member']
	last_read_seq_no bigint [not null, default: 0]
	unread_count integer [not null, default: 0]
	joined_at timestamp [not null]
	updated_at timestamp [not null]
	
	Note: 'Участие пользователя в чате и его состояние внутри чата'
	
	Indexes {
		(chat_id, user_id) [pk]
		user_id [name: 'idx_chat_members_user_id']
		(chat_id, role) [name: 'idx_chat_members_chat_role']
	}
}

  

Table messages {
	id uuid [pk, not null]
	chat_id uuid [not null]
	sender_id uuid [not null]
	seq_no bigint [not null]
	body text [not null]
	is_edited boolean [not null, default: false]
	is_encrypted boolean [not null, default: false]
	created_at timestamp [not null]
	updated_at timestamp [not null]

	Note: 'История сообщений'

	Indexes {
		(chat_id, seq_no) [name: 'uq_messages_chat_seq', unique]
		sender_id [name: 'idx_messages_sender_id']
		created_at [name: 'idx_messages_created_at']
	}
}

Table update_buffer {
	id uuid [pk, not null]
	user_id uuid [not null]
	chat_id uuid [not null]
	message_id uuid [not null]
	update_type varchar(64) [not null, note: 'new_message | read_ack | edit_message | service_event']
	payload text [not null]
	created_at timestamp [not null]
	expires_at timestamp [not null]
	
	Note: 'Буфер обновлений для доставки и синхронизации'
	
	Indexes {
		user_id [name: 'idx_update_buffer_user_id']
		chat_id [name: 'idx_update_buffer_chat_id']
		message_id [name: 'idx_update_buffer_message_id']
		expires_at [name: 'idx_update_buffer_expires_at']
	}
}

Table presence_cache {
	user_id uuid [not null]
	device_id varchar(128) [not null]
	status varchar(32) [not null, note: 'online | offline | away']
	last_seen_at timestamp [not null]
	expires_at timestamp [not null]
	
	Note: 'Онлайн-статус пользователя и last_seen'
	
	Indexes {
		(user_id, device_id) [pk]
		expires_at [name: 'idx_presence_cache_expires_at']
	}
}

Table storage_files {
	id uuid [pk, not null]
	file_type varchar(32) [not null, note: 'avatar | chat_photo | service_file']
	object_key varchar(255) [not null, unique]
	size_bytes bigint [not null]
	created_at timestamp [not null]
	
	Note: 'Метаданные файлов; бинарные объекты лежат в объектном хранилище'

	Indexes {
		object_key [name: 'uq_storage_files_object_key', unique]
		file_type [name: 'idx_storage_files_file_type']
	}
}

  

Table event_log {
	id uuid [pk, not null]
	user_id uuid [not null]
	chat_id uuid
	message_id uuid
	event_type varchar(64) [not null, note: 'send_message | read_message | login | reconnect | join_chat']
	payload text [not null]
	created_at timestamp [not null]
	
	Note: 'Журнал событий системы'
	
	Indexes {
		user_id [name: 'idx_event_log_user_id']
		chat_id [name: 'idx_event_log_chat_id']
		message_id [name: 'idx_event_log_message_id']
		created_at [name: 'idx_event_log_created_at']
	}
}

  

Ref: users.avatar_file_id > storage_files.id
Ref: user_sessions.user_id > users.id

Ref: chats.owner_id > users.id
Ref: chats.chat_photo_file_id > storage_files.id
Ref: chats.last_message_id > messages.id

Ref: chat_members.chat_id > chats.id
Ref: chat_members.user_id > users.id

Ref: messages.chat_id > chats.id
Ref: messages.sender_id > users.id

Ref: update_buffer.user_id > users.id
Ref: update_buffer.chat_id > chats.id
Ref: update_buffer.message_id > messages.id

Ref: presence_cache.user_id > users.id

Ref: event_log.user_id > users.id
Ref: event_log.chat_id > chats.id
Ref: event_log.message_id > messages.id
```

