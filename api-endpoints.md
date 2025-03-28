# API эндпоинты для тестирования

Документация по API эндпоинтам для работы с тестами.

## Основные эндпоинты для тестирования

### Прохождение тестов

#### POST /api/user/tests/:id/start
Начало тестирования, возвращает информацию о тесте и вопросах.

**Параметры пути:**
- `id` - ID теста

**Возвращает:**
```json
{
  "id": 2,
  "title": "Тест по строительным нормам и правилам",
  "description": "Проверка знаний основных строительных норм и правил",
  "time_limit": 600,
  "start_date": "2025-01-01T00:00:00.000Z",
  "end_date": "2025-12-31T23:59:59.000Z",
  "questions": [
    {
      "id": 42,
      "questionText": "Какой нормативный документ регламентирует правила производства и приемки земляных работ?",
      "explanation": "СНиП 3.02.01-87 является основным документом, регламентирующим земляные работы",
      "options": [
        {
          "id": 156,
          "text": "СНиП 3.02.01-87"
        },
        {
          "id": 157,
          "text": "ГОСТ 5180"
        },
        {
          "id": 158,
          "text": "СП 45.13330"
        },
        {
          "id": 159,
          "text": "СНиП 12-01-2004"
        }
      ]
    },
    // остальные вопросы...
  ]
}
```

#### POST /api/user/tests/:id/questions/:questionId/answer
Отправка ответа на конкретный вопрос.

**Параметры пути:**
- `id` - ID теста
- `questionId` - ID вопроса

**Тело запроса:**
```json
{
  "selectedOptions": [156] // Массив ID выбранных вариантов ответа
}
```

**Возвращает:**
```json
{
  "success": true
}
```

#### GET /api/user/tests/:id/progress
Получение прогресса прохождения теста.

**Параметры пути:**
- `id` - ID теста

**Возвращает:**
```json
{
  "totalQuestions": 10,
  "answeredQuestions": 5,
  "remainingTime": 300, // в секундах
  "progress": 50 // процент
}
```

#### POST /api/user/tests/:id/submit
Завершение теста и отправка всех ответов.

**Параметры пути:**
- `id` - ID теста

**Возвращает:**
```json
{
  "success": true,
  "resultId": 123
}
```

#### GET /api/user/tests/:id/results
Получение результатов теста.

**Параметры пути:**
- `id` - ID теста

**Возвращает:**
```json
{
  "id": 123,
  "testId": 2,
  "userId": 42,
  "score": 80,
  "maxScore": 100,
  "percentage": 80,
  "passed": true,
  "startedAt": "2025-03-26T04:30:00.000Z",
  "completedAt": "2025-03-26T04:40:00.000Z",
  "answers": [
    {
      "questionId": 42,
      "userAnswer": [156],
      "correct": true,
      "pointsEarned": 10
    },
    // остальные ответы...
  ]
}
```

### Управление тестами

#### GET /api/user/tests/available
Получение доступных для прохождения тестов.

**Возвращает:**
```json
[
  {
    "id": 2,
    "title": "Тест по строительным нормам и правилам",
    "description": "Проверка знаний основных строительных норм и правил",
    "time_limit": 600,
    "questionsCount": 10,
    "hasActiveSession": false,
    "hasPassed": false
  },
  // другие доступные тесты...
]
```

## Особенности использования API

1. **Авторизация:**
   - Все API эндпоинты для тестирования требуют авторизации
   - Используется middleware `authMiddleware` для проверки токена
   - Токен передается в заголовке Authorization: `Bearer <token>`

2. **Обработка ошибок:**
   - При ошибке авторизации возвращается статус 401
   - При ошибке доступа возвращается статус 403
   - При отсутствии ресурса возвращается статус 404
   - При ошибке в запросе возвращается статус 400
   - При внутренней ошибке сервера возвращается статус 500 с деталями ошибки

3. **Логирование:**
   - Все критические ошибки логируются на сервере
   - Для отладки можно использовать подробное логирование

## Типичные ошибки и их решение

1. **Ошибка 500 при старте теста**:
   - Проверить существование теста
   - Убедиться, что тест содержит вопросы
   - Проверить правильность полей в SQL-запросах (order_num вместо question_order)

2. **Ошибка сохранения ответа**:
   - Проверить существование активной сессии тестирования
   - Убедиться, что вопрос принадлежит этому тесту
   - Проверить формат передаваемых данных (массив ID вариантов ответа)

3. **Ошибка при получении результатов**:
   - Убедиться, что тест завершен
   - Проверить, что пользователь имеет доступ к результатам