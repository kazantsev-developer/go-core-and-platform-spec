## Пакет context в Go

**Пакет `context`** предоставляет типизированный инструмент для управления жизненным циклом выполнения операций. Он используется для каскадной передачи сигналов отмены (cancellation), установки дедлайнов (deadlines/timeouts) и проброса изолированных метаданных (request-scoped values) сквозь границы API и дерево вызовов горутин.

Основная цель `context` — предотвращение утечек горутин (goroutine leaks) и своевременное освобождение системных ресурсов (I/O, сетевые сокеты, дескрипторы БД) при долгоживущих или зависших операциях.

---

### Внутреннее устройство (Под капотом)

Интерфейс `context.Context` описан в файле `src/context/context.go` и состоит из четырех методов:

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool) // Возвращает время завершения операции (если задано)
    Done() <-chan struct{}                   // Возвращает канал-сигнал. Закрывается, когда контекст отменен
    Err() error                              // Возвращает причину отмены (Canceled или DeadlineExceeded)
    Value(key any) any                       // Ищет и возвращает значение по ключу вверх по дереву
}
```

Контексты организуются в **направленное дерево** (иммутабельный связный список), где дочерний контекст хранит ссылку на родительский (`parent`). При отмене родительского контекста сигнал каскадно распространяется на все дочерние узлы вниз по дереву.

#### Основные типы под капотом:

- **`emptyCtx`** (через `Background()` и `TODO()`): пустая структура (тип `int`), методы возвращают `nil` или `ok == false`. Используется как корень дерева.
- **`cancelCtx`** (через `WithCancel()`): содержит мьютекс для защиты полей, карту `children` для каскадной отмены дочерних контекстов и канал `done`, который закрывается при отмене.
- **`timerCtx`** (через `WithTimeout()` и `WithDeadline()`): встраивает в себя `cancelCtx` и добавляет структуру `time.Timer` для автоматического вызова отмены по таймеру.
- **`valueCtx`** (через `WithValue()`): содержит поля `key` и `val` одного типа, а также ссылку на `parent`. Метод `Value(key)` ищет ключ у себя; если не находит — рекурсивно поднимается к родителю.

---

### Правила и ограничения (Best Practices)

1. **Сигнатура функций:** Контекст всегда передается самым первым аргументом функции. Общепринятое имя переменной — `ctx` (`func Do(ctx context.Context, ...)`).
2. **Запрет на хранение в структурах:** Нельзя сохранять `Context` внутрь полей структур. Исключение — структура `http.Request`, которая исторически инкапсулирует контекст.
3. **Обязательный вызов cancel():** Функция `cancel()`, возвращаемая фабриками (`WithCancel`, `WithTimeout`), должна вызываться всегда (обычно через `defer cancel()`). Иначе внутренние таймеры рантайма или ссылки в карте `children` не будут удалены, что вызовет утечку памяти.
4. **Ограничения `WithValue`:** Предназначен исключительно для транзита метаданных запроса (TraceID, RequestID, данные аутентификации). Запрещено использовать для передачи опциональных параметров функций или глобальных зависимостей (баз данных, логгеров). Ключи должны быть уникального, неэкспортируемого типа (`type key struct{}`), чтобы исключить коллизии между сторонними библиотеками.

---

### Примеры кода

#### 1. Каскадный таймаут HTTP-запроса

```go
package main

import (
	"context"
	"fmt"
	"net/http"
	"time"
)

func executeRequest(ctx context.Context, url string) {
	// Привязываем жизненный цикл запроса к контексту
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
	if err != nil {
		fmt.Printf("ошибка создания запроса: %v\n", err)
		return
	}

	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		// При таймауте вернется ошибка: context.DeadlineExceeded
		fmt.Printf("ошибка выполнения: %v\n", err)
		return
	}
	defer resp.Body.Close()

	fmt.Printf("Успех. Статус: %s\n", resp.Status)
}

func main() {
	// Создаем контекст с лимитом выполнения в 1.5 секунды
	ctx, cancel := context.WithTimeout(context.Background(), 1500*time.Millisecond)
	defer cancel() // Освобождает ресурсы таймера рантайма

	executeRequest(ctx, "https://typicode.com")
}
```

#### 2. Безопасная передача scoped-данных через кастомный тип ключа

```go
package main

import (
	"context"
	"fmt"
)

// Кастомный неэкспортируемый тип для ключа защищает от коллизий в мапе контекста
type ctxKeyTraceID struct{}

func logWithTrace(ctx context.Context, message string) {
	// Извлечение значения: поиск идет вверх по иерархии структуры valueCtx
	if traceID, ok := ctx.Value(ctxKeyTraceID{}).(string); ok {
		fmt.Printf("[TraceID: %s] %s\n", traceID, message)
		return
	}
	fmt.Printf("[No Trace] %s\n", message)
}

func main() {
	rootCtx := context.Background()

	// Помещаем Trace ID в контекст
	ctx := context.WithValue(rootCtx, ctxKeyTraceID{}, "req_abc123")

	logWithTrace(ctx, "Бизнес-логика выполнена успешно")
}
```
