### **Таблица users**

| Поле                      | Тип данных   | Описание                           | Индексы      |
| ------------------------- | ------------ | ---------------------------------- | ------------ |
| `id`                      | SERIAL (PK)  | Уникальный идентификатор           | PK           |
| `email`                   | VARCHAR(100) | Электронная почта (уникальная)     | UNIQUE INDEX |
| `password`                | VARCHAR(50)  | Пароль                             | –            |
| `created_at` 				| TIMESTAMP    | Дата создания аккаунта             | -            |
| `last_login`              | TIMESTAMP    | Время последнего захода на сайт    | -            |
| `telegram_link`           | VARCHAR(100) | Ссылка на телеграм аккаунт         | -            |

---

### **Таблица chats**

| Поле           | Тип данных   | Описание                   | Индексы |
| -------------- | ------------ | -------------------------- | ------- |
| `id`           | SERIAL (PK)  | Уникальный идентификатор   | PK      |
| `title`        | VARCHAR(100) | Название чата              | INDEX   |
| `user_id`      | INT (FK)     | Пользоветель → `users.id`  | INDEX   |

---

### **Таблица messages**

| Поле        | Тип данных  | Описание                                 | Индексы |
| ----------- | ----------- | ---------------------------------------- | ------- |
| `id`        | SERIAL (PK) | Уникальный идентификатор                 | PK      |
| `text`      | TEXT        | Текст сообщения                          | –       |
| `chat_id`   | INT (FK)    | Чат → `chats.id`                         | INDEX   |
| `video_url` | TEXT        | Сгенерированное видел нейросетью         | -       |

---

### **Таблица media**

| Поле      | Тип данных   | Описание                   | Индексы |
| --------- | ------------ | -------------------------- | ------- |
| `id`      | SERIAL (PK)  | Уникальный идентификатор   | PK      |
| `name`    | VARCHAR(100) | Имя/путь фото              | INDEX   |
| `user_id` | INT (FK)     | Владелец фото → `users.id` | INDEX   |

---

### **Таблица importance**

| Поле       | Тип данных  | Описание                 | Индексы      |
| ---------- | ----------- | ------------------------ | ------------ |
| `id`       | SERIAL (PK) | Уникальный идентификатор | PK           |
| `user_id`  | INT (FK)    | Пользователь сайта       | INDEX        |
| `file_type`| VARCHAR(10) | Проверка на тип файла    | PK           |
| `file_url` | TEXT        | URL на исходные файлы    | PK           |


# Формат данных для взаимодействия с AI-сервисом

### Логика работы

1.  **Пользователь отправляет сообщение**: Через endpoint /messages (POST),
    сервер сохраняет сообщение в messages с role = \'user\'.

2.  **Сервер формирует промпт**:

    1.  Извлекает историю чата

    2.  Добавляет системный промпт ( на основе media из телеграмм
        канала)

    3.  Собирает JSON-запрос

3.  **Отправка к AI**: Сервер отправляет JSON-запрос к AI-сервису.

4.  **Получение ответа**: Сервер парсит ответ, сохраняет его в messages с
    role = \'assistant\', и возвращает клиенту
	
	
## Формат отправляемых данных (Запрос к AI-сервису)

Запрос отправляется как HTTP POST на endpoint AI-сервиса . Тело запроса — JSON-объект.

### Пример JSON-запроса

```json
{
    "model": "model_name",
    "messages": [
        {
            "role": "system",
            "content": "You are Brainrot AI, a helpful assistant for creating reels. Use the media files and text messages from telegram channel to select the mood and themes"
        },
        {
            "role": "user",
            "content": "Previous user message 1"
        },
        {
            "role": "assistant",
            "content": "Previous assistant response 1"
        },
        // ... (вся история чата из таблицы messages, чередующаяся user/assistant)
        {
            "role": "user",
            "content": "Current user message"
        }
    ],
    "max_tokens": ?,
    "temperature": 0.7,
}
```

### Описание полей

model: Строка, указывающая на нашу LLM модель

messages: Массив объектов, где каждый объект --- сообщение:

-   role: \'system\' (для системного промпта, только первый), \'user\'(сообщения пользователя), \'assistant\' (предыдущие ответы модели).

-   content: Текст сообщения. Для system --- шаблон с плейсхолдерами {name}, {events_list} и т.д., заполняемыми из БД (users и events).

max_tokens: Целое число, ограничение на использования токенов в сообщении

temperature: Число (0-1), контролирует случайность (0 --- детерминировано, 1 --- креативно).


## Формат получаемых данных (Ответ от AI-сервиса)


### Пример JSON-ответа

```
{
    "id": "",
    "created": ,
    "model": " model_name",
    "choices": [
        {
            "index": 0,
            "message": {
                "role": "assistant",
                "content": {
                    "summary_text": "Your week was full of joy and travel. Here's a short highlight reel suggestion.",
                    "mood": "energetic",
                    "music_suggestion": "Upbeat electronic track",
                    "scenes": [
                        {
                            "media_url": "https://cdn.server.com/media/photo1.jpg",
                            "caption": "Morning coffee in the city",
                        },
                        {
                            "media_url": "https://cdn.server.com/media/photo2.jpg",
                            "caption": "Evening walk by the sea",
                        }
                    ],
                    "video_url": "https://cdn.server.com/videos/highlight-week-123.mp4"
                }
            },
            "finish_reason": "stop"
        }
    ],
    "usage": {
        "prompt_tokens": 210,
        "completion_tokens": 90,
        "total_tokens": 300
    }
}

```

### Описание полей

- id: Строка, уникальный идентификатор ответа от AI-сервиса (можно сохранять в логах для последующей отладки и анализа)
- choices: Массив объектов, содержащий один или несколько вариантов ответов модели (чаще всего один)
  - index: Порядковый номер варианта (0, если единственный ответ)
  - message: Объект с сообщением, которое сгенерировала модель
    - role: 'assistant'
    - content: Объект с данными, сформированными AI-моделью
      - summary_text: Краткое описание недели пользователя или общий сюжет видео
      - mood: Эмоциональный тон подборки (например, "calm", "happy", "energetic")
      - music_suggestion: Предложение музыкального сопровождения, основанное на настроении недели
      - scenes: Массив сцен, используемых для видеомонтажа
        - media_url: Ссылка на исходное изображение или видеофайл из Telegram-канала
        - caption: Подпись или краткое описание сцены (может использоваться как титр)
      - video_url: Ссылка на готовый видеофайл (MP4) после сборки видеоролика сервером
  - finish_reason: Строка, причина завершения генерации ("stop", "length", "content_filter" и т.д.)
- usage: Объект с метриками использования токенов
  - prompt_tokens: Количество токенов, использованных в исходном промпте (включая историю чата и системный контекст)
  - completion_tokens: Количество токенов, затраченных на генерацию текущего ответа модели
  - total_tokens: Общее количество токенов (prompt + completion), используется для мониторинга производительности и стоимости запроса
- created: Целое число (Unix timestamp) — время генерации ответа в формате timestamp
- model: Строка, указывает название модели, с помощью которой был сгенерирован ответ

