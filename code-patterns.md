# Шаблоны кода и рекомендации для DGB Testing

Рекомендации и шаблоны кода для работы с проектом DGB Testing.

## Работа с SQL-запросами

### Правильный способ получения вопросов для теста

```typescript
// Получаем связанные вопросы с оптимизированным запросом
const questionsResult = await pool.query(`
  SELECT 
    q.id, q.question_text as "questionText", q.explanation
  FROM test_questions tq
  JOIN questions q ON tq.question_id = q.id
  WHERE tq.test_id = $1
  ORDER BY tq.order_num
`, [id]);
```

### Безопасная обработка вопросов с вариантами ответов

```typescript
// Для каждого вопроса получаем варианты ответов
const questions = [];

for (const question of questionsResult.rows) {
  try {
    const optionsResult = await pool.query(`
      SELECT 
        id, text, is_correct as "isCorrect"
      FROM question_options
      WHERE question_id = $1
    `, [question.id]);
    
    if (optionsResult.rows.length === 0) {
      console.log(`[UserTestsController] Внимание: Вопрос ${question.id} не имеет вариантов ответа`);
    }
    
    // Перемешиваем варианты ответов и скрываем правильные ответы
    const options = optionsResult.rows
      .map(o => ({ ...o, isCorrect: undefined }))
      .sort(() => Math.random() - 0.5);
    
    questions.push({
      ...question,
      options
    });
  } catch (error) {
    console.error(`[UserTestsController] Ошибка при получении вариантов ответа для вопроса ${question.id}:`, error);
    // Продолжаем выполнение, пропуская этот вопрос
  }
}

// Проверка наличия вопросов после обработки
if (questions.length === 0) {
  return res.status(500).json({ 
    error: 'Не удалось загрузить вопросы теста',
    details: 'Вопросы не найдены или не имеют вариантов ответа'
  });
}
```

### Обработка конфликтов при создании/обновлении записей

```typescript
// Создаем или обновляем запись о начале прохождения теста
try {
  await pool.query(`
    INSERT INTO test_sessions (user_id, test_id, started_at, status)
    VALUES ($1, $2, CURRENT_TIMESTAMP, 'in_progress')
    ON CONFLICT (user_id, test_id) WHERE status = 'in_progress'
    DO UPDATE SET started_at = CURRENT_TIMESTAMP, updated_at = CURRENT_TIMESTAMP
  `, [userId, testId]);
  
  console.log(`[UserTestsController] Создана/обновлена сессия тестирования для пользователя ${userId}, тест ${testId}`);
} catch (sessionError) {
  console.error(`[UserTestsController] Ошибка при создании сессии тестирования:`, sessionError);
  // Продолжаем выполнение даже при ошибке с сессией, так как основные данные уже получены
}
```

## React компоненты и хуки

### Хук для управления состоянием теста

```tsx
export function useTestSession(testId: number) {
  const [session, setSession] = useState<TestSession | null>(null);
  const [currentQuestionIndex, setCurrentQuestionIndex] = useState(0);
  const [answers, setAnswers] = useState<Record<number, number[]>>({});
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);
  
  // Инициализация сессии
  const initSession = async () => {
    try {
      setLoading(true);
      const response = await api.startTest(testId);
      setSession(response);
      setLoading(false);
    } catch (error) {
      console.error('[TestSession] Error initializing session:', error);
      setError('Ошибка при инициализации теста');
      setLoading(false);
    }
  };
  
  // Сохранение ответа
  const saveAnswer = async (questionId: number, selectedOptions: number[]) => {
    try {
      // Обновляем локальное состояние
      setAnswers(prev => ({
        ...prev,
        [questionId]: selectedOptions
      }));
      
      // Отправляем ответ на сервер
      await api.saveAnswer(testId, questionId, selectedOptions);
      
      return true;
    } catch (error) {
      console.error('[TestSession] Error saving answer:', error);
      return false;
    }
  };
  
  // Остальные методы...
  
  return {
    session,
    loading,
    error,
    currentQuestionIndex,
    currentQuestion: session?.questions[currentQuestionIndex] || null,
    answers,
    initSession,
    saveAnswer,
    nextQuestion: () => setCurrentQuestionIndex(prev => Math.min(prev + 1, (session?.questions.length || 1) - 1)),
    prevQuestion: () => setCurrentQuestionIndex(prev => Math.max(prev - 1, 0)),
    // Остальные методы...
  };
}
```

### Компонент тестовой сессии

```tsx
export function TestSession() {
  const { id } = useParams<{ id: string }>();
  const testId = parseInt(id || '0');
  const navigate = useNavigate();
  
  const {
    session,
    loading,
    error,
    currentQuestion,
    currentQuestionIndex,
    answers,
    initSession,
    saveAnswer,
    nextQuestion,
    prevQuestion,
    submitTest
  } = useTestSession(testId);
  
  // Инициализация при монтировании
  useEffect(() => {
    console.log('[TestSessionDebug] Initializing test session', testId);
    initSession().catch(err => {
      console.error('[TestSessionDebug] Test session error', err);
    });
  }, [testId]);
  
  // Обработчик ответа
  const handleAnswer = async (selectedOptions: number[]) => {
    if (!currentQuestion) return;
    
    const success = await saveAnswer(currentQuestion.id, selectedOptions);
    if (success) {
      toast.success('Ответ сохранен');
      nextQuestion();
    } else {
      toast.error('Ошибка при сохранении ответа');
    }
  };
  
  // Обработчик завершения теста
  const handleSubmit = async () => {
    const success = await submitTest();
    if (success) {
      toast.success('Тест успешно завершен');
      navigate(`/tests/${testId}/results`);
    } else {
      toast.error('Ошибка при завершении теста');
    }
  };
  
  // Отображение компонента
  if (loading) return <Spinner />;
  if (error) return <ErrorMessage message={error} />;
  if (!session) return <EmptyState message="Тест не найден" />;
  
  return (
    <div className="test-session">
      {/* Содержимое компонента */}
    </div>
  );
}
```

## Обработка ошибок и логирование

### Логирование на сервере

```typescript
// Логирование с контекстом
console.log(`[UserTestsController] Начало теста ${id} для пользователя ${userId}`);

// Логирование ошибок
try {
  // Код, который может вызвать ошибку
} catch (error) {
  console.error('[UserTestsController] Ошибка при начале теста:', error);
  res.status(500).json({
    error: 'Ошибка при начале теста',
    details: error instanceof Error ? error.message : 'Неизвестная ошибка'
  });
}
```

### Логирование на клиенте

```typescript
// Логирование API ошибок
api.interceptors.response.use(
  response => response,
  error => {
    // Логирование ошибок API
    if (error.response) {
      console.error(
        `[API] Ошибка ${error.response.status} при выполнении ${error.config.method} ${error.config.url}`,
        `Параметры запроса: ${error.config.params}`,
        `Тело запроса: ${error.config.data}`,
        `Ответ сервера: ${error.response.data}`
      );
    } else {
      console.error(`[API] Ошибка сети: ${error.message}`);
    }
    
    return Promise.reject(error);
  }
);
```

## Советы по производительности

1. **Оптимизация запросов:**
   - Избегайте сложных JOIN и агрегаций в одном запросе
   - Разделяйте сложные запросы на более простые

2. **Оптимизация React компонентов:**
   - Используйте React.memo для предотвращения ненужных рендеров
   - Используйте useCallback для функций, передаваемых в дочерние компоненты

3. **Кэширование данных:**
   - Используйте React Query для кэширования серверных данных
   - Применяйте оптимистичные обновления для улучшения UX