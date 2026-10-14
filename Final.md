Понял. Тогда задача формулируется иначе — нужна **одна большая UML-диаграмма классов** (`classDiagram`), но сгруппированная по компонентам через **packages** или **namespaces**. В Mermaid это делается через `namespace` внутри `classDiagram`.

## Задача для агента: сводная UML-диаграмма классов

### Контекст

Есть проектная документация shell'а на Go, разбитая по слоям. Каждый слой описан отдельным файлом с текстовым описанием и собственной UML-диаграммой классов (`classDiagram` в Mermaid):

- `Lexer.md` — лексический анализатор;
- `Parser.md` — построение AST;
- `Expander.md` — раскрытие переменных (сервис, вызываемый executor'ом);
- `Executor.md` — обход AST и исполнение;
- `CommandsAndRegistry.md` — интерфейс `Command` и реестр;
- `Environment.md` — таблица переменных окружения;
- `ExecContext.md` — потоки и окружение одной команды;
- `REPL.md` — `Shell` как владелец REPL-цикла и сборщик системы.

Архитектурные соглашения — в `Instructions.md`: соответствие Go-типов UML-типам (`[]T` → `T [*]`, `map[K]V` → `Dict(K, V)`, `io.Reader` → `InputStream`, `io.Writer` → `OutputStream`).

### Что нужно сделать

Собрать **одну сводную UML-диаграмму классов** вместо восьми отдельных. Диаграмма — **`classDiagram`** в Mermaid. Никаких `flowchart`.

### Группировка по компонентам

Использовать **namespace** внутри `classDiagram`, чтобы сгруппировать классы по компонентам:

```mermaid
classDiagram
    namespace Lexer {
        class Lexer
        class LexError
    }
    namespace Parser {
        class Parser
        class ParseError
    }
    namespace AST {
        class Node
        class SimpleCommand
        class Pipeline
        class Assign
    }
    namespace Expander {
        class Expander
    }
    namespace Executor {
        class Executor
    }
    namespace Commands {
        class Command
        class Registry
        class CatCommand
        class EchoCommand
        class WcCommand
        class PwdCommand
        class ExitCommand
    }
    namespace Shell {
        class Shell
    }
    namespace Environment {
        class Environment
    }
    namespace Context {
        class ExecContext
    }
    namespace Shared {
        class Word
        class WordPart
        class Token
        class QuoteKind
        class TokenType
    }
```

Если Mermaid-версия не поддерживает `namespace` — использовать **комментарии-разделители** (`%% ===== Lexer =====`) и располагать классы группами.

### Требования к содержимому классов

**Только ключевые члены.** Полные сигнатуры — в детальных диаграммах. На сводной показывать:

- имя типа;
- 1–3 ключевых поля;
- 1–3 ключевых метода (без полных сигнатур).

Пример:

```
class Expander {
    +ExpandArgs(words : Word[*]) : String[*]
    +ExpandAssignValue(w : Word) : String
}
```

### Интерфейсы и DI

Соблюдать **Dependency Inversion** там, где это осмысленно. Ввести интерфейсы:

| Интерфейс       | Реализация                       | Кто зависит |
| --------------- | -------------------------------- | ----------- |
| `Tokenizer`     | `Lexer`                          | `Shell`     |
| `Parsing`       | `Parser`                         | `Shell`     |
| `Executing`     | `Executor`                       | `Shell`     |
| `Expanding`     | `Expander`                       | `Executor`  |
| `CommandLookup` | `Registry`                       | `Executor`  |
| `Command`       | `CatCommand`, `EchoCommand`, ... | `Registry`  |
| `Node`          | `SimpleCommand`, `Pipeline`      | `Executor`  |

Связь «реализует» — пунктирная стрелка с полым треугольником: `Lexer ..|> Tokenizer`.

**DTO не прячутся за интерфейсами.** `Word`, `Token`, `ExecContext`, `Environment` — конкретные классы. Это осознанное решение, обосновать в пояснении.

### Общие типы

`Word`, `WordPart`, `Token`, `QuoteKind`, `TokenType`, `Node` используются несколькими компонентами. Вынести их в отдельный namespace **Shared** или **AST** (для `Node` и наследников). Это снимает необходимость тянуть стрелки из всех углов диаграммы.

### Связи

**Типы связей** — стандартные UML:

- `*--` — композиция (владеет, часть целого);
- `o--` — агрегация (содержит, но не владеет);
- `-->` — ассоциация (знает о, использует);
- `..>` — зависимость (использует временно);
- `<|--` — наследование;
- `<|..` — реализация интерфейса.

**Направление связей** — от владельца к зависимому, сверху вниз по слоям:

- `Shell` владеет `Environment`, `Registry`, `Lexer`, `Parser`, `Executor`;
- `Shell` использует интерфейсы `Tokenizer`, `Parsing`, `Executing`;
- `Executor` использует `Expanding`, `CommandLookup`, обходит `Node`;
- `Executor` создаёт `ExecContext`;
- `Expander` читает `Word`, `Environment`;
- `Registry` содержит `Command`;
- команды читают `ExecContext`.

### Расположение

Mermaid сам располагает классы, но **порядок объявления** влияет на компоновку. Объявлять классы **сверху вниз по слоям**:

1. внешние границы (`Main`, `OS`, `ExecCmd`);
2. `Shell`;
3. конвейер (`Lexer`, `Parser`, `Executor`);
4. сервисы (`Expander`, `Registry`, `Environment`);
5. команды (`Command` и реализации);
6. контекст (`ExecContext`);
7. общие типы (`Word`, `Token`, `Node`).

Mermaid обычно рисует ортогонально внутри `classDiagram` — специально просить «прямые углы» не нужно.

### Что НЕ нужно

- Не использовать `flowchart` или `graph`.
- Не дублировать полные сигнатуры из детальных диаграмм.
- Не вводить интерфейсы над DTO (`Word`, `Token`, `Environment`, `ExecContext`).
- Не показывать `pipePair`, работу с `os.Pipe()`, коды возврата — детали реализации.
- Не детализировать стандартные команды — оставить их как отдельные классы, но без членов (только `+Name()`, `+Run()`).

### Ожидаемый результат

1. **Mermaid-диаграмма** (`classDiagram`), готовая к вставке.
2. **Текст-пояснение** (10–20 строк):
   - почему выбрана такая группировка;
   - какие интерфейсы введены и почему;
   - почему DTO не спрятаны за интерфейсами;
   - как на диаграмме отражены границы с внешним миром.

### Критерии готовности

- Все компоненты представлены: `Lexer`, `Parser`, `Expander`, `Executor`, `Shell`, `Environment`, `ExecContext`, `Registry`, `Commands`.
- Интерфейсы `Tokenizer`, `Parsing`, `Executing`, `Expanding`, `CommandLookup`, `Command`, `Node` присутствуют.
- Общие типы (`Word`, `Token`, `Node`) вынесены в отдельный namespace.
- Связи — стандартные UML-стрелки с правильными типами (`*--`, `o--`, `..>`, `<|..`).
- Диаграмма читается без детальных `*.md`.
