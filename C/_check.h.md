**Check** это фреймворк модульного (unit) тестирования для языка C. Он подключается как обычная библиотека (`check.h` + `-lcheck`) и даёт:

- макросы для описания тестов (`START_TEST` / `END_TEST`);
- набор проверок-утверждений (`ck_assert_*`);
- структуру для группировки тестов (suite, test case);
- запускатор тестов (`SRunner`) с отчётом о результатах.

В C нет встроенной тестовой инфраструктуры, как в Python или Java. Без неё приходится писать `main` с кучей `if` и `printf`. Check убирает эту рутину и делает тесты воспроизводимыми и читаемыми.

---

**Ключевая особенность: изоляция через `fork`**

По умолчанию каждый тест запускается в **отдельном процессе** (`fork`). Это даёт три вещи:

- Segmentation fault в одном тесте не роняет весь набор. Check фиксирует ошибку и идёт к следующему тесту.
- Тесты не влияют друг на друга: глобальное состояние и порча памяти остаются в своём процессе.
- Можно проверять, что функция **должна** упасть с сигналом или завершиться с определённым кодом.

>Минус: отладка через `gdb` и `valgrind` в режиме fork неудобна, поэтому для неё `fork` отключают.

# Установка и сборка

```bash
# Debian / Ubuntu
sudo apt install check

# macOS (Homebrew)
brew install check
```

Самый надёжный способ узнать нужные флаги это `pkg-config`:

```bash
pkg-config --cflags --libs check
```

Типичная сборка:

```bash
gcc -Wall -Werror -Wextra -std=c11 tests.c s21_string.a -o test $(pkg-config --cflags --libs check)
```

На Linux для Check обычно нужны дополнительные библиотеки: `-lcheck -lsubunit -lrt -lm -lpthread`. `pkg-config` подставит их сам. Если его нет, перечисляйте вручную. Порядок важен: библиотеки линкуются **после** исходников.


# Компоненты для организации тестов

```
SRunner                 запускает и собирает результаты
  └── Suite             логический модуль (файл)
        └── TCase       группа тестов с общими настройками ("core")
              └── Test  одна проверка (START_TEST ... END_TEST)
```

| Сущность | Тип                         | Назначение                                              |
| -------- | --------------------------- | ------------------------------------------------------- |
| Test     | `START_TEST` ... `END_TEST` | одна проверка или сценарий                              |
| TCase    | `TCase *`                   | группа связанных тестов с общими фикстурами и таймаутом |
| Suite    | `Suite *`                   | набор групп, обычно по модулю или функции               |
| SRunner  | `SRunner *`                 | запускает наборы и собирает результат                   |
Сборка снизу вверх: пишем тесты, кладём их в `TCase`, `TCase` в `Suite`, `Suite` в `SRunner`, запускаем и читаем результат.

## Тест: `START_TEST` / `END_TEST`

```c
START_TEST(strlen_1) {
  ck_assert_uint_eq(s21_strlen("hello"), strlen("hello"));
}
END_TEST
```

Это макросы. Они разворачиваются в `static void strlen_basic(int _i) { ... }`.

|Элемент|Назначение|
|---|---|
|`START_TEST(name)`|Объявляет тест с именем `name`. Это функция без возвращаемого значения.|
|`END_TEST`|Закрывает функцию, обязателен.|
|`_i`|Скрытый параметр `int`. Для обычных тестов не нужен, для циклических (`tcase_add_loop_test`) это текущий индекс.|

Тест **проходит**, если дошёл до конца без сработавшего `ck_assert*`. Тест **падает**, если сработал `ck_assert*` или `ck_abort`, программа получила сигнал (SIGSEGV, SIGABRT), истёк таймаут или процесс завершился через `exit()` без ожидания этого.

### Как прерывается тест

Это зависит от режима запуска:

- **`CK_FORK` (по умолчанию).** Для **каждого** теста Check делает `fork()`. Тест выполняется в дочернем процессе, а родитель ждёт и читает результат через канал (pipe). Если в тесте случился segfault, процесс умирает, родитель видит сигнал и записывает "тест упал по SIGSEGV", а остальные тесты продолжают выполняться.
- **`CK_NOFORK`.** Всё в одном процессе, провал реализован через `longjmp`. Падение по сигналу убьёт всю программу.

Из режима fork есть важное практическое следствие: **каждый тест работает с чистой копией памяти родителя**. Изменили глобальную переменную в одном тесте, в следующем она снова будет исходной. Тесты не могут влиять друг на друга через глобальное состояние, и утечка памяти в одном тесте не накапливается. Но и передать что-то от теста к тесту через глобальную переменную тоже нельзя.


## `TCase`: группа тестов

```c
TCase *tc = tcase_create("core");   // имя группы, строка

tcase_add_test(tc, strlen_1);
tcase_add_test(tc, strlen_2);

suite_add_tcase(s, tc);             // прикрепить группу к набору
```

`TCase` в памяти это структура, которая хранит:

1. список тестов (адрес и имя каждого);
2. параметры, общие для всех тестов группы: таймаут, фикстуры, теги;
3. имя группы.

>Освобождать `TCase` вручную не нужно: им владеет `Suite`, а `Suite` владеет `SRunner`. Вызов `srunner_free` очищает всю цепочку.


```c
// Пример деления на группы для s21_sprintf

Suite *sprintf_suite(void) {
  Suite *s = suite_create("s21_sprintf");

  TCase *ints = tcase_create("integers");
  tcase_add_test(ints, sprintf_d_positive);
  tcase_add_test(ints, sprintf_d_negative);
  tcase_add_test(ints, sprintf_d_width);

  TCase *strs = tcase_create("strings");
  tcase_add_test(strs, sprintf_s_basic);
  tcase_add_test(strs, sprintf_s_precision);

  TCase *errors = tcase_create("edge_cases");
  tcase_add_test(errors, sprintf_null_format);
  tcase_add_test(errors, sprintf_percent_at_end);

  suite_add_tcase(s, ints);
  suite_add_tcase(s, strs);
  suite_add_tcase(s, errors);
  return s;
}
```

### Зачем нужны группы

**1. Общая подготовка и очистка (фикстуры)**: Фикстура выполняется перед каждым тестом группы и после него.

```c
static char *buf;

static void setup(void) {
  buf = malloc(64);
  memset(buf, 'X', 64);       // заполняем мусором, чтобы увидеть недописанный '\0'
}

static void teardown(void) {
  free(buf);
}

Suite *strcpy_suite(void) {
  Suite *s = suite_create("s21_strcpy");
  TCase *tc = tcase_create("core");

  tcase_add_checked_fixture(tc, setup, teardown);
  tcase_add_test(tc, strcpy_basic);
  tcase_add_test(tc, strcpy_empty);

  suite_add_tcase(s, tc);
  return s;
}
```

Внутри тестов используется `buf` без повторного `malloc`. Без группы пришлось бы копировать выделение и очистку в начало и конец каждого теста. Переменные фикстуры должны быть глобальными или `static` на уровне файла, потому что `setup` не принимает параметров и ничего не возвращает: общаться с тестом ему больше не через что.


**2. Разные настройки для разных видов тестов**: настройки привязаны к группе, а не к отдельному тесту. Если бы такой группировки не было, пришлось бы выбирать один таймаут для всего Suite.

```c
TCase *fast = tcase_create("fast");        // таймаут по умолчанию (4 с)

TCase *heavy = tcase_create("big_buffers");
tcase_set_timeout(heavy, 30);              // для тяжёлых тестов
```


**3. Выборочный запуск при отладке**

```
CK_RUN_CASE=big_buffers ./test
```

Запустятся только тесты группы с таким именем, во всех Suite. Для сравнения: `CK_RUN_SUITE` отбирает по набору. Когда у `s21_sprintf` 200 тестов, а отлаживаете вы только работу с флагами, такая выборка экономит время. Отдельный тест из группы запустить нельзя, это минимальная единица фильтрации, поэтому группы стоит делать по смыслу.


**4. Читаемый отчёт**: в сообщении о провале есть имя группы. `s21_sprintf:flags:test_plus_flag` говорит больше, чем просто `test_plus_flag`.


### Функции групп

#### Макрос `tcase_add_test()`

Добавляет обычный тест. Написать `START_TEST` недостаточно: функция сама никуда не регистрируется. Её нужно явно добавить в группу:

```c
tcase_add_test(tc, fn);
```

Макрос `tcase_add_test` берёт имя функции, превращает его в строку (через `#`) и сохраняет вместе с указателем.

```c
TCase* tc = tcase_create("core");

tcase_add_test(tc, strlen_1);
tcase_add_test(tc, strlen_2);
tcase_add_test(tc, strlen_3);
tcase_add_test(tc, strlen_4);
```


#### Макрос `tcase_add_loop_test()`

```c
tcase_add_loop_test(tc, fn, start, end)
```

Запускает тест в цикле для `_i = start ... end-1`. Каждая итерация это отдельный запуск.

```c
START_TEST(test_digits) {
  char buf[2] = {'0' + _i, '\0'};
  ck_assert_uint_eq(s21_strlen(buf), 1);
}
END_TEST

tcase_add_loop_test(tc, test_digits, 0, 10);  // _i = 0..9
```


#### Функции для добавления фикстур

```c
void tcase_add_checked_fixture(TCase *tc, void (*setup)(void), void (*teardown)(void));

void tcase_add_unchecked_fixture(TCase *tc, void (*setup)(void), void (*teardown)(void));
```

| Параметр                     | **checked**                                         | **unchecked**                                         |
| ---------------------------- | --------------------------------------------------- | ----------------------------------------------------- |
| Когда выполняется            | **перед и после каждого теста**                     | **один раз** для всего TCase                          |
| Где выполняется              | в дочернем процессе вместе с тестом (в режиме fork) | в основном процессе                                   |
| Если внутри упал `ck_assert` | считается ошибкой теста                             | не отслеживается, ломает весь запуск                  |
| Типичное применение          | выделить и освободить буфер, подготовить структуру  | создать временный файл, инициализировать общий ресурс |


#### `tcase_set_timeout()`


```c
void tcase_set_timeout(TCase *tc, double timeout)
```

Лимит времени на **каждый** тест. По умолчанию 4 секунды. `0` означает без лимита (в новых версиях).


## `Suite`: набор групп тестов

### Тип данных `Suite`

В `check.h` это непрозрачный тип (opaque type). Само определение структуры спрятано внутри библиотеки. Поля вам недоступны, работать с `Suite` можно только через функции Check. Всегда используется указатель `Suite *`, объект создаётся в куче внутри библиотеки.

Это **именованный упорядоченный список TCase**. Больше он ничего не хранит и ничего не делает. Сам тесты не запускает, настроек (таймаут, фикстуры) не имеет, их несёт `TCase`.

#### Освобождение памяти

Память Suite освобождает `srunner_free`, причём вместе со всеми его TCase и тестами. Отсюда два правила:

1. Suite, не переданный ни в один `SRunner`, **утечёт**.
2. Один и тот же `Suite` или `TCase` нельзя добавлять в два раннера или два набора: будет двойное освобождение.


### Функции для работы с типом

#### `suite_create`

```c
Suite *suite_create(const char *name);
```

- **Вход:** имя набора. Строка не копируется, поэтому передавайте литерал или строку, которая живёт дольше раннера.
- **Выход:** новый пустой `Suite *`.
- **Имя нужно для:**
    - строки `Running suite(s): s21_strlen` в начале отчёта;
    - фильтра `CK_RUN_SUITE=s21_strlen`;
    - `srunner_run(sr, "s21_strlen", NULL, mode)`.


#### `suite_add_tcase`

```c
void suite_add_tcase(Suite *s, TCase *tc);
```

- **Вход:** набор и группа тестов.
- **Выход:** ничего.
- Группа добавляется в конец списка. Порядок запуска равен порядку добавления. Тесты внутри `TCase` тоже идут в порядке добавления.


### Функция-сборщик (builder)

```c
Suite *strlen_suite(void) {
  Suite *s = suite_create("s21_strlen");
  TCase *tc = tcase_create("core");

  tcase_add_test(tc, strlen_1);
  tcase_add_test(tc, strlen_2);
  suite_add_tcase(s, tc);

  return s;
}
```

`START_TEST` разворачивается в **`static` функцию**. Из другого `.c` файла на тест сослаться нельзя. Значит, регистрировать тесты (`tcase_add_test`) обязан код **в том же файле**, где они написаны. Отсюда схема:

```
test_strlen.c   тесты (static) + strlen_suite()   ← регистрирует их у себя
test_string.h   прототип: Suite *strlen_suite(void);
test_string.c   main(): видит только сборщик, тесты ему не нужны
```

Сборщик это единственная "публичная" точка файла с тестами. Для `main` весь набор это просто вызов функции, возвращающей `Suite *`.

#### Правила хорошего сборщика

- Возвращает указатель, созданный через `suite_create`. Никогда не возвращайте указатель на локальную переменную.
- Один сборщик, один Suite, одна тестируемая функция (или группа родственных).
- Имя Suite делайте равным имени тестируемой функции (`"s21_strlen"`), так проще запускать выборочно и читать отчёт.
- Новый TCase добавляйте только когда нужны **другие** настройки (таймаут, фикстуры).

#### Типичный вариант с несколькими TCase

```c
Suite *sprintf_suite(void) {
  Suite *s = suite_create("s21_sprintf");

  TCase *tc_int = tcase_create("integers");
  tcase_add_test(tc_int, sprintf_d_basic);
  tcase_add_test(tc_int, sprintf_d_width);

  TCase *tc_big = tcase_create("big_input");
  tcase_set_timeout(tc_big, 20);
  tcase_add_test(tc_big, sprintf_long_string);

  suite_add_tcase(s, tc_int);
  suite_add_tcase(s, tc_big);
  return s;
}
```


## `SRunner`: запуск тестов

```c
#include "test_string.h"

int main(void) {
  Suite *suites[] = {strlen_suite(), strcpy_suite(), strcmp_suite()};
  size_t n = sizeof(suites) / sizeof(suites[0]);

  SRunner *sr = srunner_create(NULL);
  for (size_t i = 0; i < n; i++) srunner_add_suite(sr, suites[i]);

  srunner_set_log(sr, "test.log");     // до запуска
  srunner_run_all(sr, CK_NORMAL);

  int failed = srunner_ntests_failed(sr);
  srunner_free(sr);                    // освободит и Suite'ы

  return failed ? EXIT_FAILURE : EXIT_SUCCESS;
}
```

### Тип данных `SRunner`

`SRunner` (Suite Runner) нужен, чтобы **запускать** наборы тестов и **хранить результаты** прогона. Сами тесты, TCase и Suite только описывают, что проверять. Выполняет проверки и собирает итог раннер.

```
Suite ─┐
Suite ─┼──► SRunner ──► запуск ──► результаты (счётчики, список TestResult, отчёты)
Suite ─┘
```

**Жизненный цикл**

```c
SRunner *sr = srunner_create(first_suite);   // 1. создать
srunner_add_suite(sr, second_suite);         // 2. добавить ещё Suite
srunner_set_fork_status(sr, CK_NOFORK);      // 3. (опц.) настроить
srunner_set_log(sr, "test.log");             //    (опц.) вести лог
srunner_run_all(sr, CK_NORMAL);              // 4. запустить
int failed = srunner_ntests_failed(sr);      // 5. прочитать результат
srunner_free(sr);                            // 6. освободить
```

Настройки (fork, логи) нужно задавать **до** запуска, иначе они не подействуют.


### Функции для работы с типом

#### `srunner_create()`

```c
SRunner *sr(Suite *s);
```

Создаёт и возвращает `SRunner*`, принимает либо один `Suite*`, либо `NULL`. С `NULL` получится пустой раннер, Suite'ы добавляются потом в цикле:


```c
Suite* tests[] = {strlen_suite(), strcat_suite(), memchr_suite()};
size_t items = sizeof(tests) / sizeof(tests[0]);

SRunner* sr = srunner_create(NULL);
for (size_t i = 0; i < items; i++) srunner_add_suite(sr, tests[i]);
```


#### `srunner_add_suite()`

```c
void srunner_add_suite(SRunner *sr, Suite *s);
```

Добавляет `Suite*` в `SRunner`. Можно вызывать сколько угодно раз. Suite'ы выполняются в порядке добавления.

>`Suite` после передачи раннеру принадлежит ему. Отдельно `free` не нужен, иначе получится двойное освобождение.


#### `srunner_run_all()`

```c
void srunner_run_all(SRunner *sr, enum print_output print_mode);
```

Запускает все тесты, добавленные в `SRunner`

##### Режимы вывода

|Константа|Поведение|
|---|---|
|`CK_SILENT`|Ничего не печатает. Результаты читаются программно.|
|`CK_MINIMAL`|Только итоговая строка.|
|`CK_NORMAL`|Итог и подробности по упавшим тестам. Лучший вариант по умолчанию.|
|`CK_VERBOSE`|Строка по каждому тесту, включая успешные.|
|`CK_ENV`|Режим берётся из переменной `CK_VERBOSITY` (`silent`, `minimal`, `normal`, `verbose`).|

Итоговая строка:

```
100%: Checks: 4, Failures: 0, Errors: 0
```

Строка об ошибке имеет формат `файл:строка:тип:TCase:тест:итерация:сообщение`:

```
test_strlen.c:14:F:core:strlen_4:0: Assertion 's21_strlen(NULL) == 0' failed
```

Тип бывает таким:

- **`F` (Failure)**: сработала проверка `ck_assert*` или `ck_abort`.
- **`E` (Error)**: тест «сломался» иначе: сигнал (SIGSEGV, SIGABRT), таймаут, неожиданный `exit()`.


#### `srunner_set_fork_status()`

```c
void srunner_set_fork_status(SRunner *sr, enum fork_status fstat);
```

|Константа|Поведение|
|---|---|
|`CK_FORK_GETENV`|Значение по умолчанию. Смотрит на переменную `CK_FORK`: `no` отключает fork, всё остальное включает.|
|`CK_FORK`|Всегда fork: каждый тест в отдельном процессе.|
|`CK_NOFORK`|Все тесты в одном процессе.|

| Параметр                                              | **`CK_FORK`**                                 | **`CK_NOFORK`**               |
| ----------------------------------------------------- | --------------------------------------------- | ----------------------------- |
| Изоляция                                              | Падение теста не убивает прогон               | Segfault роняет всю программу |
| Таймауты                                              | Работают                                      | Не работают                   |
| `tcase_add_test_raise_signal`, `tcase_add_exit_value` | Работают                                      | Не работают                   |
| Отладка в gdb                                         | Неудобно (нужно `set follow-fork-mode child`) | Удобно                        |
| valgrind                                              | Шумно, много процессов                        | Чисто, нагляднее              |
| Скорость                                              | Медленнее из-за `fork()` на каждый тест       | Быстрее                       |

>Для обычного прогона и `make test` оставляйте `fork`, а для **gdb** и **valgrind** запускайте так: `CK_FORK=no ./test` (если `CK_FORK_GETENV`) или явно `CK_NOFORK`.


#### Получение результатов

##### `srunner_ntests_run()`

```c
int srunner_ntests_run(SRunner *sr);
```

Возвращает число выполненных тестов (циклические тесты считаются по итерациям).


##### `srunner_ntests_failed()`

```c
int srunner_ntests_failed(SRunner *sr);
```

Возвращает число тестов с результатом `F` или `E`. Это значение идёт в код возврата


#### `srunner_free()`

```c
void srunner_free(SRunner *sr);
```

Освобождает сам раннер, все его `Suite`, `TCase` и результаты. После `srunner_free` указатели на Suite, TCase и `TestResult*` недействительны. Результаты, прочитанные из раннера, нужно использовать до его освобождения.


# Утверждения

| Что проверяем                       | Макрос                               |
| ----------------------------------- | ------------------------------------ |
| Длина, количество (`size_t`)        | `ck_assert_uint_eq`                  |
| Код возврата, `int`, `char`, `enum` | `ck_assert_int_eq`                   |
| Знак результата `strcmp`/`memcmp`   | `ck_assert_int_eq(SIGN(a), SIGN(b))` |
| Строка, не может быть `NULL`        | `ck_assert_str_eq`                   |
| Строка, может быть `NULL`           | `ck_assert_pstr_eq`                  |
| Адрес в исходной строке             | `ck_assert_ptr_eq`                   |
| Результат `NULL` или нет            | `ck_assert_ptr_null` / `_nonnull`    |
| Буфер, бинарные данные              | `ck_assert_mem_eq`                   |
| `double` после вычислений           | `ck_assert_double_eq_tol`            |
| `inf`/`nan`                         | `ck_assert_double_infinite` / `_nan` |
| Сложное условие, свой текст         | `ck_assert_msg`                      |
| Недостижимая ветка                  | `ck_abort_msg`                       |

`ck_assert*` это **макросы**, а не функции. Каждый макрос делает три вещи:

1. Вычисляет аргументы и сравнивает их.
2. Если проверка не прошла, формирует сообщение (файл, строка, выражение, фактические значения).
3. **Сразу прерывает текущий тест** и регистрирует его как упавший.

Поэтому внутри одного теста выполняется только до первого неудачного assert. Остальные проверки после него уже не запустятся.

```c
START_TEST(example) {
  ck_assert_int_eq(1, 1);   // прошёл, идём дальше
  ck_assert_int_eq(2, 3);   // упал, тест закончился здесь
  ck_assert_int_eq(4, 4);   // никогда не выполнится
}
END_TEST
```


### Базовые проверки

```c
ck_assert(expr);
ck_assert_msg(expr, fmt, ...);
ck_abort();
ck_abort_msg(fmt, ...);
```

|Макрос|Вход|Поведение|
|---|---|---|
|`ck_assert(expr)`|любое выражение, ненулевое считается истиной|падает, если `expr == 0`. Сообщение: `Assertion 'expr' failed`|
|`ck_assert_msg(expr, fmt, ...)`|то же плюс формат в стиле `printf`|при падении печатает **ваше** сообщение вместо стандартного|
|`ck_abort()`|ничего|безусловное падение (например, в ветке, куда код не должен попасть)|
|`ck_abort_msg(fmt, ...)`|формат и аргументы|то же с сообщением|

Когда использовать `ck_assert_msg`: если стандартного сообщения мало, например внутри цикла по набору данных.

```c
for (int i = 0; i < n; i++) {
  ck_assert_msg(s21_strlen(cases[i]) == strlen(cases[i]),
                "failed on case #%d: \"%s\"", i, cases[i]);
}
```

Тут `ck_assert_uint_eq` показал бы только числа, а не то, на какой строке из набора сломалось.


### Целые числа

Шаблон: `ck_assert_<тип>_<операция>(X, Y)`.

|Тип|Внутри приводится к|Использовать для|
|---|---|---|
|`int`|`intmax_t`|`int`, `char`, `short`, `long`, `enum`, результат `strcmp`|
|`uint`|`uintmax_t`|`unsigned`, **`size_t`**, `uint8_t` и т.п.|

|Операция|Условие|
|---|---|
|`_eq`|`X == Y`|
|`_ne`|`X != Y`|
|`_lt`|`X < Y`|
|`_le`|`X <= Y`|
|`_gt`|`X > Y`|
|`_ge`|`X >= Y`|

```c
ck_assert_uint_eq(s21_strlen("abc"), 3);
ck_assert_int_ne(s21_strncmp("a", "b", 1), 0);
ck_assert_uint_lt(count, capacity);
```


>**Сравнение `strcmp`/`memcmp` с `s21_` версией напрямую**: стандарт гарантирует лишь знак результата, а не конкретное число. Одна реализация вернёт `-1`, другая `-98`. Поэтому сравнивайте знаки:

```c
#define SIGN(x) (((x) > 0) - ((x) < 0))
ck_assert_int_eq(SIGN(s21_strcmp(a, b)), SIGN(strcmp(a, b)));
```


### Вещественные числа

Типы: `float`, `double`, `ldouble` (long double). Появились в Check 0.11.

#### Точное сравнение

```c
ck_assert_double_eq(X, Y);   // также _ne, _lt, _le, _gt, _ge
```

Это обычное `==` над `double`. Для результатов вычислений оно почти всегда **не подходит** из-за ошибок округления.

#### Сравнение с допуском

```c
ck_assert_double_eq_tol(X, Y, T);   // |X - Y| <= T
ck_assert_double_ne_tol(X, Y, T);   // |X - Y| >  T
ck_assert_double_ge_tol(X, Y, T);   // X >= Y - T
ck_assert_double_le_tol(X, Y, T);   // X <= Y + T
```

`T` это допустимая абсолютная погрешность. Для `float` и `long double` аналогично: `ck_assert_float_eq_tol`, `ck_assert_ldouble_eq_tol`.

#### Проверка особых значений

```c
ck_assert_double_finite(X);     // не inf и не nan
ck_assert_double_infinite(X);   // +inf или -inf
ck_assert_double_nan(X);        // nan
ck_assert_double_nonnan(X);     // не nan
```

Особенно полезно для `s21_sprintf`/`s21_sscanf` с `%f`, `%e`, `%g`: отдельные тесты на `inf` и `nan`.

```c
double x;
s21_sscanf("3.14", "%lf", &x);
ck_assert_double_eq_tol(x, 3.14, 1e-9);
```


### Строки

```c
ck_assert_str_eq(X, Y);   // также _ne, _lt, _le, _gt, _ge
ck_assert_pstr_eq(X, Y);  // также _pstr_ne
```

|Семейство|Сравнивает|`NULL`|
|---|---|---|
|`ck_assert_str_*`|как `strcmp`|**падает с ошибкой**, если любой аргумент `NULL`|
|`ck_assert_pstr_*`|как `strcmp`, но «безопасно»|допускается: `NULL == NULL` истинно, `NULL` и непустая строка различны|

Семейство `pstr_*` (pointer-safe) нужно именно для функций, которые могут вернуть `NULL`, например `s21_strstr`, `s21_strtok`, `s21_strpbrk`, `s21_strerror` при некорректном вводе:

```c
ck_assert_pstr_eq(s21_strstr("abc", "zzz"), strstr("abc", "zzz"));  // оба NULL
```

При падении Check печатает обе строки в кавычках, так что видны пробелы и пустые строки.

**Чего `str_*` не делает:** не сравнивает по длине и не знает про `\0` внутри буфера. Для бинарных данных используйте `mem_*`.


### Указатели

```c
ck_assert_ptr_eq(X, Y);      // адреса равны
ck_assert_ptr_ne(X, Y);      // адреса различны
ck_assert_ptr_null(X);       // X == NULL
ck_assert_ptr_nonnull(X);    // X != NULL
```

Сравнивается **адрес**, а не содержимое. Идеально для функций, возвращающих указатель в исходную строку:

```c
const char *s = "hello world";
ck_assert_ptr_eq(s21_strchr(s, 'w'), strchr(s, 'w'));
ck_assert_ptr_null(s21_strchr(s, 'z'));
```

Тут важно, что оба вызова работают с **одним и тем же** `s`. Если передавать разные копии строки, адреса будут разными даже при правильной реализации.


### Память

```c
ck_assert_mem_eq(X, Y, L);   // также _ne, _lt, _le, _gt, _ge
```

|Параметр|Смысл|
|---|---|
|`X`, `Y`|указатели на блоки памяти|
|`L`|сколько байт сравнивать (`size_t`)|

Работает как `memcmp(X, Y, L)`. Применять для `memcpy`, `memmove`, `memset`, `memchr`, а также для проверки, что буфер после операции не затронут за пределами записи.

```c
char dst1[10] = {0}, dst2[10] = {0};
s21_memcpy(dst1, "abc", 4);
memcpy(dst2, "abc", 4);
ck_assert_mem_eq(dst1, dst2, sizeof(dst1));   // сравнили весь буфер, включая «хвост»
```

При падении Check печатает **hex-дамп** обоих блоков. Для коротких блоков это удобно.

---

### 8. Метка `mark_point()`

c

```c
mark_point();
```

Запоминает файл и строку. Если тест позже упадёт **сигналом** (SIGSEGV) или по таймауту, то в отчёте будет указана последняя пройденная метка. Без неё Check показывает только «тест упал по сигналу» без строки.

c

```c
START_TEST(crash_hunt) {
  mark_point();
  s21_function_a();
  mark_point();
  s21_function_b();    // если упадёт здесь, в отчёте будет строка второй метки
}
END_TEST
```

Метки работают только в режиме `CK_FORK`.

---

### 9. Где можно вызывать ck_assert

|Место|Можно?|
|---|---|
|Тело `START_TEST`|да|
|Вспомогательная функция, вызванная из теста|да (в той же ветке выполнения)|
|`checked` фикстура (`setup`/`teardown`)|да|
|`unchecked` фикстура|нет, сломает весь запуск|
|`main` вне раннера|нет|
|Другой поток (`pthread`)|нет, не потокобезопасно|

Особенности режимов:

- **`CK_FORK`** (по умолчанию): после падения процесс теста завершается. Память, которую тест не успел освободить, ушла вместе с процессом. Valgrind при этом показывает утечки только в упавших тестах.
- **`CK_NOFORK`**: прерывание делается через `longjmp`, **освобождающий код после assert не выполняется**. Утечки вы увидите из-за самого факта падения.

Практическое правило: если в тесте есть `malloc`, то лучше освободить память **до** assert или хранить буферы в checked-фикстуре, чтобы `teardown` выполнился даже при падении:

c

```c
// плохо: при падении free не вызовется
char *r = s21_insert("abc", "X", 1);
ck_assert_str_eq(r, "aXbc");
free(r);

// лучше: сначала сохраняем результат сравнения, потом освобождаем, потом assert
char *r = s21_insert("abc", "X", 1);
int ok = (r != NULL && strcmp(r, "aXbc") == 0);
free(r);
ck_assert(ok);
```

Для `s21_to_upper`, `s21_insert`, `s21_trim` (часть 5 вашего проекта) это актуально.