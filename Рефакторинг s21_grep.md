Ниже проверка по вашим принципам: C11/gcc, POSIX.1-2017, Google Style, структурное программирование, без дублирования, без глобальных переменных, один выход из функции и цикла, вложенность до 4, функция до 40–50 строк.

Сразу хорошее: `goto` и глобальных переменных нет, выходы из циклов и функций в основном единые, `_POSIX_C_SOURCE 200809L` корректен для POSIX.1-2017, устаревших функций (`legacy`/`obsolete`) нет, форматирование похоже на clang-format Google.

## 1. Критические проблемы

**1.1. `fclose(NULL)` в `read_pattern_file`.** Если `fopen` не открыл файл, цикл пропускается, но `fclose(fp)` всё равно вызывается. Команда `s21_grep -f нет_такого_файла x` упадёт с UB (на glibc обычно segfault). Исправление: закрывать файл только в ветке успешного открытия.

```c
Status LoadPatternFile(RawPatterns* patterns, const char* path) {
  Status status = kStatusOk;
  FILE* file = fopen(path, "r");
  if (file == NULL) {
    perror(path);
    status = kStatusError;
  } else {
    char* line = NULL;
    size_t capacity = 0;
    ssize_t length;
    while (status == kStatusOk && (length = getline(&line, &capacity, file)) != -1) {
      if (line[length - 1] == '\n') line[length - 1] = '\0';
      status = AddRawPattern(patterns, line);
    }
    free(line);
    fclose(file);
  }
  return status;
}
```

**1.2. Ошибка в `-l` вместе с `-v`.** Условие выхода из цикла в `process_file` содержит лишний фрагмент `(invert_match && was_match_global)`. `was_match_global` накапливается по всем файлам. После первого же выбранного строкой файла для каждого следующего файла цикл завершится после первой строки, даже если она не выбрана. В результате `-lv foo a b` может не вывести `b`. Достаточно условия `show_only_filenames && matchs_count > 0`, потому что `was_match` уже инвертирован.

**1.3. `-o` с несколькими шаблонами выводит только один шаблон.** В `multi_matcher` цикл идёт с последнего шаблона и останавливается на первом, который совпал (`!was_match`). Для `-o -e foo -e bar` на строке `foo bar` выведется только `bar`, а GNU grep выведет `foo` и `bar`. Нужно на каждой позиции курсора искать самое левое совпадение среди всех шаблонов (при равенстве начала берётся самое длинное):

```c
static bool FindLeftmostMatch(const RegexList* regexes, const char* text,
                              int eflags, regmatch_t* best) {
  bool found = false;
  for (size_t i = 0; i < regexes->count; ++i) {
    regmatch_t current;
    if (regexec(&regexes->items[i], text, 1, &current, eflags) == 0 &&
        (!found || current.rm_so < best->rm_so ||
         (current.rm_so == best->rm_so && current.rm_eo > best->rm_eo))) {
      *best = current;
      found = true;
    }
  }
  return found;
}
```

## 2. Серьёзные проблемы

- **`-o -v`.** Сейчас печатаются целые строки, а GNU grep не печатает ничего.
- **UB при работе с указателями.**
    - В `match_all_in_line` `cursor += rm_eo + 1` может уйти за `end + 1`, а это уже UB. Работайте со смещениями `size_t` и сравнивайте их с длиной строки.
    - В `count_patterns` при пустом `-f` файле `data == NULL`, и вычисляется `NULL + 0`, что тоже UB в C.
- **Границы слов при `-o`.** Поиск с подстрокой `cursor` и `REG_NOTBOL` может давать неверный результат для `\<`, `\b`. Корректное решение — `REG_STARTEND`, но это расширение, не POSIX. Проще задокументировать ограничение.
- **Ошибки чтения не обрабатываются.** Цикл `getline` не отличает EOF от ошибки (`ferror`). Для директории `fopen` успешен, и grep молча ничего не делает, а GNU пишет `Is a directory`. Ошибки записи в stdout тоже не проверяются: нужен `ferror(stdout)`/`fclose(stdout)` и код 2.
- **Нет stdin.** Нет чтения stdin при отсутствии файлов и `-` как имени файла. Если это не требуется заданием, зафиксируйте это в документации.
- **`regcomp` без `REG_NOSUB`.** Он нужен везде, где не работает `-o`: тогда регулярка быстрее.
- **Типы счётчиков.** `int` для номера строки и числа совпадений переполнится на огромных файлах. Лучше `size_t` и `%zu`.

## 3. Нарушения ваших принципов

- **Лишние выходы из функций.**
    - В `process_file` ранний `return -1` после `fopen` — это не проверка аргументов, нужно оформить `if/else`.
    - В `print_match` три `return`.
- **Длина функций.**
    - `main` занимает около 55–60 строк с комментариями и превышает лимит.
    - `process_file` и `parse_args` на границе лимита.
- **Дублирование.**
    - В `-e` и `-f` повторяется связка `if (add...) is_error=-1; if (!is_error) was_e_or_f=true;`. Заодно можно писать `is_error = Add...()`, раз функция уже возвращает -1.
    - Путь к файлу передаётся и параметром, и через `state.file_path`.
    - `show_help` вызывается из трёх мест.
- **Смешение ответственности.** `single_matcher` и `match_all_in_line` одновременно ищут и печатают, а `process_file` дополнительно печатает для `-v`. Логику `-v` приходится восстанавливать по всему коду. Правильнее чистый предикат `LineMatches`, затем `selected = matched != invert`, затем вывод.
- **`set_pattern`.** Разбирает операнд и ещё выставляет `more_than_one_file`.
- **`count_patterns`.** Лишняя функция: счётчик шаблонов проще вести при добавлении в `add_raw_pattern`.
- **`print_match`.** Ветка `else` с `printf("%s\n")` недостижима. Для `len == -1` срабатывает `if (len)`, а `%.*s` с отрицательной точностью по стандарту печатает всю строку. Это работает случайно, и «магическое» -1 лучше убрать: передавайте `strlen`.

## 4. Мелкие замечания

- В `show_help` склейка строк без пробела даёт `[-hnlcovsi]{ PATTERN`.
- `show_help` вызывается дважды, если `getopt` вернул `?` и `optind == argc`. Проверка `first_nonflag >= argc` в `main` не защищена `!is_error`.
- Параметр `optarg` в `add_raw_pattern` и `read_pattern_file` затеняет глобальную переменную `getopt` (`-Wshadow`).
- Прототип `parse_args` называет параметр `patterns_s`, определение — `raw_patterns_s`.
- В `case '?'` нет `break`, в `switch` нет `default`.
- Смешаны сообщения об ошибках на русском и английском. Выберите один язык (для утилиты лучше английский).
- Нарушено правило Google о скобках: в `count_patterns` `if (nl) { ... } else exit_cycle = true;`. Если одна ветка со скобками, вторая тоже.
- `char* p` там, где нужно `const char*` (`count_patterns`, `file_path`).
- Магические числа 0/1/2 для кодов возврата.
- Нет `#include <stddef.h>` в `types.h` (`size_t` приходит транзитивно через `regex.h`).
- Нет сообщения об ошибке при `calloc` в `compline_patterns`.
- Объявление `int reti;` стоит далеко от места использования.
- Опечатки в именах: `compline_patterns`, `matchs_count`.

Флаги компиляции для отлова многого из этого: `-std=c11 -Wall -Wextra -Wpedantic -Wshadow -Wconversion -Wstrict-prototypes -Wmissing-prototypes -fsanitize=address,undefined`. Определение `_POSIX_C_SOURCE` можно перенести в Makefile через `-D`.

## 5. Именование по Google C++ Style, адаптированное под C

|Сущность|Правило|Пример|
|---|---|---|
|Файлы|`snake_case`, расширения `.c`/`.h`|`options.c`, `search.h`|
|Типы (struct/enum/typedef)|`PascalCase`, без суффиксов `_t` и `_s`|`Options`, `RegexList`|
|Функции|`PascalCase`, начинаются с глагола|`CompilePatterns`, `SearchFile`|
|Переменные и параметры|`snake_case`, без типовых суффиксов|`raw_patterns`, `match_count`|
|Поля структур|`snake_case` без подчёркивания в конце (оно только у членов классов C++)|`line_number`|
|Константы|`k` + `PascalCase`; целочисленные в C лучше через `enum`|`kMinCapacity`|
|Enum-значения|`kName`|`kStatusOk`, `kExitNoMatch`|
|Макросы|избегать; если нужны, то `UPPER_SNAKE` с префиксом проекта|`S21_GREP_NAME`|
|Header guard|`PROJECT_PATH_FILE_H_`|`S21_GREP_SRC_OPTIONS_H_`|

Дополнительно:

- Булевы флаги называйте утвердительно. `no_filename` и `no_messages` дают двойные отрицания вроде `!opts->no_filename`, лучше `suppress_filename`, `suppress_errors`.
- В `Options` флаг `line_number` и счётчик `state.line_number` называются одинаково. Переименуйте флаг в `show_line_numbers`.
- Нужны точные имена: `old_len` — это указатель, `nl` в `count_patterns` ищет `\0`, `reti` взято из примера man.

Предлагаемые переименования:

|Было|Стало|
|---|---|
|`show_help`|`PrintUsage`|
|`parse_args`|`ParseCommandLine`|
|`set_pattern`|`ResolvePatternOperand`|
|`add_raw_pattern`|`AddRawPattern`|
|`read_pattern_file`|`LoadPatternFile`|
|`compline_patterns`|`CompilePatterns`|
|`free_regex_list`|`FreeRegexList`|
|`ensure_capacity`|`ReserveCapacity`|
|`single_matcher`, `multi_matcher`, `match_all_in_line`|`LineMatches`, `PrintOnlyMatching`|
|`print_match`|`PrintLine`|
|`show_counting_results`|`PrintFileSummary`|
|`process_file`|`SearchFile`|
|`set_exit_code`|`ComputeExitCode`|
|`exit_cycle`|`done`|
|`reti`|`status`|
|`line_buf`, `nread`|`line`, `line_length`|
|`matchs_count`|`match_count`|
|`was_e_or_f`|`has_pattern_option`|
|`was_match_global`|`any_line_selected`|
|`was_error_global`|`any_file_error`|
|`more_than_one_file`|`multiple_files`|
|`MIN_ALLOC_BUF_CAP`|`kMinCapacity`|
|`S21`|`kProgramName`|
|`show_counting`|`count_only`|
|`show_only_filenames`|`list_files_only`|
|`show_only_matching`|`only_matching`|

## 6. Структуры и прототипы

Опции после разбора не меняются, поэтому состояние выносится отдельно: опции передаются как `const Options*`. Состояние, которое живёт весь запуск, отделяется от состояния одного файла. Так функции вывода и сопоставления перестают изменять опции.

```c
/* options.h */
typedef enum { kStatusOk = 0, kStatusError = -1 } Status;

typedef struct Options {
  bool ignore_case, invert_match, count_only, list_files_only;
  bool show_line_numbers, suppress_filename, suppress_errors, only_matching;
  bool has_pattern_option;   /* был -e или -f */
  bool multiple_files;
} Options;

Status ParseCommandLine(int argc, char* const argv[], Options* options,
                        RawPatterns* patterns, int* first_file_index);
```

```c
/* patterns.h */
typedef struct RawPatterns {
  char* data;        /* шаблоны, разделённые '\0' */
  size_t length, capacity, count;
} RawPatterns;

typedef struct RegexList {
  regex_t* items;
  size_t count;
} RegexList;

Status AddRawPattern(RawPatterns* patterns, const char* text);
Status LoadPatternFile(RawPatterns* patterns, const char* path);
Status CompilePatterns(const RawPatterns* raw, const Options* options,
                       RegexList* regexes);
void FreeRawPatterns(RawPatterns* patterns);
void FreeRegexList(RegexList* regexes);
```

```c
/* output.h */
typedef struct FileState {
  const char* path;
  size_t line_number, match_count;
} FileState;

void PrintLine(const Options* options, const FileState* file,
               const char* text, size_t length);
void PrintFileSummary(const Options* options, const FileState* file);
```

```c
/* search.h */
typedef struct RunState {
  bool any_line_selected, any_file_error;
} RunState;

Status SearchFile(const char* path, const RegexList* regexes,
                  const Options* options, RunState* run);
```

```c
/* main.c */
typedef enum { kExitMatch = 0, kExitNoMatch = 1, kExitError = 2 } ExitCode;

int main(int argc, char* argv[]);
static int RunGrep(int argc, char* const argv[]);
static ExitCode ComputeExitCode(Status status, const RunState* run);
```

Внутренние (`static`) функции по файлам:

```c
/* options.c */
static void PrintUsage(void);
static Status HandleOption(int opt, Options* options, RawPatterns* patterns);
static Status ResolvePatternOperand(int argc, char* const argv[],
                                    int* first_file_index, Options* options,
                                    RawPatterns* patterns);

/* patterns.c */
static Status ReserveCapacity(RawPatterns* patterns, size_t needed);
static size_t SplitByNewlines(char* start, size_t length);

/* search.c */
static bool LineMatches(const RegexList* regexes, const char* line);
static bool FindLeftmostMatch(const RegexList* regexes, const char* text,
                              int eflags, regmatch_t* best);
static void PrintOnlyMatching(const char* line, const RegexList* regexes,
                              const Options* options, const FileState* file);
static void ProcessLine(const char* line, const RegexList* regexes,
                        const Options* options, FileState* file, RunState* run);

/* output.c */
static void PrintPrefix(const Options* options, const FileState* file);
```

Ядро `ProcessLine` после рефакторинга выглядит так:

```c
bool matched = LineMatches(regexes, line);
bool selected = (matched != options->invert_match);
if (selected) {
  ++file->match_count;
  run->any_line_selected = true;
  if (options->only_matching && !options->invert_match) {
    PrintOnlyMatching(line, regexes, options, file);
  } else if (!options->count_only && !options->list_files_only) {
    PrintLine(options, file, line, strlen(line));
  }
}
```

Условие `-l` для выхода из цикла в `SearchFile` становится простым: `options->list_files_only && file->match_count > 0`.

## 7. Нужен ли `static`

Да, для всех функций, кроме `main` и тех, что составляют публичный API модуля, потому что:

- internal linkage не даёт именам `ensure_capacity`, `show_help` и т. п. конфликтовать с другими объектными файлами и libc;
- компилятор предупредит о неиспользуемых функциях (`-Wunused-function`) и лучше выбросит или встроит код;
- публичная поверхность модуля видна сразу.

Прототипы статических функций держите в начале `.c`-файла, чтобы `main` или публичные функции могли стоять первыми (это удобно для разработки сверху вниз). В заголовки попадают только публичные функции. Минус: статическую функцию нельзя вызвать из юнит-теста напрямую. Для ваших интеграционных тестов на bash это неважно. Если дойдёте до юнит-тестов (например, Check), тестируйте через публичный API модулей.

## 8. Разделять ли на файлы

Да, но умеренно. Около 450 строк в одном файле ещё терпимо, но `main`, разбор аргументов, шаблоны, поиск и вывод — пять разных ответственностей. Пять модулей достаточно, больше не нужно:

```
src/grep/
  main.c         — main, RunGrep, ComputeExitCode
  options.{c,h}  — Options, Status, ParseCommandLine, PrintUsage
  patterns.{c,h} — RawPatterns, RegexList, добавление, загрузка из файла, компиляция
  search.{c,h}   — SearchFile, сопоставление строки, RunState
  output.{c,h}   — FileState, PrintLine, PrintFileSummary
Makefile         — цели s21_grep, tests, clean
tests/           — data/, скрипт сравнения с grep/ggrep
```

Общий `types.h` лучше убрать. Монолитный заголовок со всеми типами связывает все модули друг с другом. Каждый тип живёт в заголовке своего модуля, а зависимости идут в одну сторону: `main → search → output, patterns → options`. Отдельный `Buffer` не нужен, пока динамический буфер используется только для шаблонов: он остаётся полями `RawPatterns`.

Если хотите, могу следующим шагом написать полный рефакторенный код по этой структуре.