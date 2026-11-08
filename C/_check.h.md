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


#### `srunner_free()`



