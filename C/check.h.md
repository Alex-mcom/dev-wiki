**Check** это фреймворк модульного (unit) тестирования для языка C. Он подключается как обычная библиотека (`check.h` + `-lcheck`) и даёт:

- макросы для описания тестов (`START_TEST` / `END_TEST`);
- набор проверок-утверждений (`ck_assert_*`);
- структуру для группировки тестов (suite, test case);
- запускатор тестов (`SRunner`) с отчётом о результатах.

В C нет встроенной тестовой инфраструктуры, как в Python или Java. Без неё приходится писать `main` с кучей `if` и `printf`. Check убирает эту рутину и делает тесты воспроизводимыми и читаемыми.

---

> [!Ключевая особенность: изоляция через fork]

По умолчанию каждый тест запускается в **отдельном процессе** (`fork`). Это даёт три вещи:

- Segmentation fault в одном тесте не роняет весь набор. Check фиксирует ошибку и идёт к следующему тесту.
- Тесты не влияют друг на друга: глобальное состояние и порча памяти остаются в своём процессе.
- Можно проверять, что функция **должна** упасть с сигналом или завершиться с определённым кодом.

Минус: отладка через `gdb` и `valgrind` в режиме fork неудобна, поэтому для неё fork отключают.

Для проектов вроде своей реализации `string.h`, `sprintf`, `sscanf` Check нужен, чтобы сравнивать вашу функцию с эталонной из libc на множестве входных данных:

```c
ck_assert_int_eq(s21_strlen(str), strlen(str));
```

## Установка и сборка

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

## Архитектура тестов

```
SRunner (запускатор)
  └── Suite (набор)
        └── TCase (группа тестов)
              └── Test (отдельный тест)
```

|Сущность|Тип|Назначение|
|---|---|---|
|Test|`START_TEST` ... `END_TEST`|одна проверка или сценарий|
|TCase|`TCase *`|группа связанных тестов с общими фикстурами и таймаутом|
|Suite|`Suite *`|набор групп, обычно по модулю или функции|
|SRunner|`SRunner *`|запускает наборы и собирает результат|

### Описание теста

```c
START_TEST(test_strlen_basic) {
  ck_assert_uint_eq(s21_strlen("hello"), strlen("hello"));
}
END_TEST
```

Макрос разворачивается в `static void имя(int _i)`. Параметр `_i` нужен для циклических тестов.

### Сборка набора

```c
Suite *strlen_suite(void) {
  Suite *s = suite_create("s21_strlen");
  TCase *tc = tcase_create("core");

  tcase_add_test(tc, test_strlen_basic);
  suite_add_tcase(s, tc);
  return s;
}
```

### Запуск

```c
int main(void) {
  Suite *s = strlen_suite();
  SRunner *sr = srunner_create(s);

  srunner_run_all(sr, CK_NORMAL);
  int failed = srunner_ntests_failed(sr);
  srunner_free(sr);

  return (failed == 0) ? EXIT_SUCCESS : EXIT_FAILURE;
}
```

Код возврата важен: по нему `make` и CI понимают, прошли ли тесты.

## Функции для организации тестов

### Работа с Suite и TCase

|Функция|Что делает|
|---|---|
|`suite_create(name)`|создаёт набор|
|`tcase_create(name)`|создаёт группу тестов|
|`suite_add_tcase(s, tc)`|добавляет группу в набор|
|`tcase_add_test(tc, fn)`|добавляет тест в группу|
|`tcase_add_loop_test(tc, fn, start, end)`|запускает тест в цикле для значений `_i` от `start` до `end - 1`|
|`tcase_set_timeout(tc, sec)`|таймаут на каждый тест группы (по умолчанию небольшой, задаётся в секундах)|
|`tcase_add_checked_fixture(tc, setup, teardown)`|setup и teardown перед и после **каждого** теста, внутри fork|
|`tcase_add_unchecked_fixture(tc, setup, teardown)`|setup и teardown один раз на группу, вне fork|

### Циклические тесты

Позволяют прогнать один тест на наборе данных без копирования кода:

```c
static const char *samples[] = {"", "a", "hello", "long string here"};

START_TEST(test_strlen_loop) {
  ck_assert_uint_eq(s21_strlen(samples[_i]), strlen(samples[_i]));
}
END_TEST

// в сборке набора:
tcase_add_loop_test(tc, test_strlen_loop, 0, 4);
```

При падении Check покажет, на каком значении `_i` это произошло.

### Тесты на аварийное завершение

Работают только в режиме fork:

|Функция|Проверяет|
|---|---|
|`tcase_add_test_raise_signal(tc, fn, signal)`|что тест завершится с указанным сигналом, например `SIGSEGV`|
|`tcase_add_exit_value(tc, fn, value)`|что процесс завершится с указанным кодом|

### Фикстуры

Фикстура это подготовка и очистка окружения теста: выделить память, создать файл, обнулить глобальные переменные.

```c
static char *buf;

static void setup(void) { buf = malloc(100); }
static void teardown(void) { free(buf); }

tcase_add_checked_fixture(tc, setup, teardown);
```

Checked-фикстура выполняется в процессе теста, поэтому утечки и падения в ней тоже ловятся.

## Утверждения (assertions)

Это основной инструмент: если условие ложно, тест помечается как проваленный и **немедленно завершается**.

### Универсальные

|Макрос|Назначение|
|---|---|
|`ck_assert(expr)`|выражение истинно|
|`ck_assert_msg(expr, fmt, ...)`|то же, но с вашим сообщением в формате `printf`|
|`ck_abort()`|безусловный провал|
|`ck_abort_msg(fmt, ...)`|то же, с сообщением|

### Целые числа

|Макрос|Тип|Операции|
|---|---|---|
|`ck_assert_int_*`|знаковые (`intmax_t`)|`eq`, `ne`, `lt`, `le`, `gt`, `ge`|
|`ck_assert_uint_*`|беззнаковые (`uintmax_t`)|те же|

Правило выбора: для `size_t` (результат `strlen`, `sizeof`) используйте `ck_assert_uint_eq`. Для `int` и `long` используйте `ck_assert_int_eq`. Иначе возможны неверные сравнения и странные значения в сообщении об ошибке.

### Строки

|Макрос|Назначение|
|---|---|
|`ck_assert_str_eq(a, b)`|строки равны (`strcmp == 0`)|
|`ck_assert_str_ne(a, b)`|строки различаются|
|`ck_assert_str_lt/le/gt/ge`|лексикографическое сравнение|

Нельзя передавать `NULL`: поведение неопределённое, возможен segfault. Сообщение при провале красиво покажет обе строки.

### Указатели

|Макрос|Назначение|
|---|---|
|`ck_assert_ptr_eq(a, b)`|указатели равны|
|`ck_assert_ptr_ne(a, b)`|указатели различаются|
|`ck_assert_ptr_null(p)`|указатель равен `NULL`|
|`ck_assert_ptr_nonnull(p)`|указатель не `NULL`|

Макросы `ck_assert_ptr_null` и `ck_assert_ptr_nonnull` появились в более новых версиях Check. На старых вместо них пишут `ck_assert_ptr_eq(p, NULL)`.

### Память

|Макрос|Назначение|
|---|---|
|`ck_assert_mem_eq(a, b, n)`|первые `n` байт совпадают (как `memcmp == 0`)|
|`ck_assert_mem_ne(a, b, n)`|не совпадают|
|`ck_assert_mem_lt/le/gt/ge`|сравнение по `memcmp`|

Применяйте для проверки `memcpy`, `memset`, `memmove` и произвольных буферов, где нет нулевого терминатора.

### Числа с плавающей точкой

В новых версиях Check есть семейства `ck_assert_float_*`, `ck_assert_double_*`, `ck_assert_ldouble_*` с операциями `eq`, `ne`, `lt`, `le`, `gt`, `ge`. Для сравнения с допуском есть вариации `_tol`:

```c
ck_assert_double_eq_tol(s21_value, expected, 1e-6);
```

Точное `eq` для вещественных почти всегда плохая идея из-за погрешностей. Для проверки `sprintf` и `sscanf` с `%f` используйте `_tol`.

### Вспомогательные

|Макрос|Назначение|
|---|---|
|`mark_point()`|запоминает место в коде, чтобы при падении Check показал последнюю достигнутую точку|

## Запуск и результаты

### Режимы подробности вывода

Передаются во второй аргумент `srunner_run_all`:

|Константа|Что выводит|
|---|---|
|`CK_SILENT`|ничего|
|`CK_MINIMAL`|только итоговую сводку|
|`CK_NORMAL`|сводку и описание провалов|
|`CK_VERBOSE`|сводку и строку по каждому тесту|
|`CK_ENV`|берёт режим из переменной окружения `CK_VERBOSITY`|

### Как читать вывод

```
100%: Checks: 12, Failures: 0, Errors: 0
```

или при проблемах:

```
tests.c:25:F:core:test_strlen_basic:0: Assertion 's21_strlen("hello") == 5' failed
tests.c:40:E:core:test_memcpy:0: (after this point) Received signal 11 (Segmentation fault)
```

Формат: `файл:строка:тип:группа:тест:значение_i: сообщение`.

- **F (Failure)**: утверждение не выполнилось. Ваша функция вернула не то.
- **E (Error)**: тест аварийно завершился (сигнал, таймаут, неожиданный код выхода). Чаще всего это segfault или бесконечный цикл.

### Полезные функции SRunner

|Функция|Назначение|
|---|---|
|`srunner_create(suite)`|создаёт запускатор|
|`srunner_add_suite(sr, s)`|добавляет ещё один набор|
|`srunner_run_all(sr, mode)`|запускает всё|
|`srunner_ntests_run(sr)`|сколько тестов запущено|
|`srunner_ntests_failed(sr)`|сколько провалено|
|`srunner_free(sr)`|освобождает ресурсы|
|`srunner_set_fork_status(sr, CK_NOFORK)`|отключает fork|
|`srunner_set_log(sr, "file.log")`|дублирует вывод в файл|
|`srunner_set_xml(sr, "file.xml")`|отчёт в формате XML|
|`srunner_set_tap(sr, "file.tap")`|отчёт в формате TAP|

### Режим без fork

```c
srunner_set_fork_status(sr, CK_NOFORK);
```

или без изменения кода:

```bash
CK_FORK=no ./test
```

Когда нужен:

- отладка в `gdb`;
- запуск под `valgrind` (проверка утечек и чтения за границами);
- диагностика, если fork мешает чему-то в окружении.

В этом режиме тесты на сигналы не работают, а падение одного теста убивает весь процесс.

```bash
CK_FORK=no valgrind --leak-check=full ./test
```

## Пример организации большого проекта

### Один файл на модуль

Обычно структура такая: по файлу на группу функций (`test_strlen.c`, `test_memcpy.c`, ...), в каждом функция `Suite *..._suite(void)`, и общий `main`:

```c
// tests.h
#ifndef TESTS_H_
#define TESTS_H_
#include <check.h>

Suite *strlen_suite(void);
Suite *memcpy_suite(void);

#endif  // TESTS_H_
```

```c
// tests.c
#include <stdlib.h>
#include "tests.h"

int main(void) {
  int failed = 0;
  Suite *suites[] = {strlen_suite(), memcpy_suite(), NULL};

  for (int i = 0; suites[i] != NULL; i++) {
    SRunner *sr = srunner_create(suites[i]);
    srunner_set_fork_status(sr, CK_FORK);
    srunner_run_all(sr, CK_NORMAL);
    failed += srunner_ntests_failed(sr);
    srunner_free(sr);
  }
  return (failed == 0) ? EXIT_SUCCESS : EXIT_FAILURE;
}
```

### Сравнение со стандартной библиотекой

Приём для проекта со своей `string.h`: каждый тест сравнивает результат вашей функции с эталоном.

```c
START_TEST(test_strncmp_cmp) {
  const char *a = "abcd";
  const char *b = "abce";
  int mine = s21_strncmp(a, b, 4);
  int ref = strncmp(a, b, 4);
  // знак результата, а не точное значение: стандарт не гарантирует конкретное число
  ck_assert_int_eq((mine > 0) - (mine < 0), (ref > 0) - (ref < 0));
}
END_TEST
```

Важная деталь: для функций сравнения (`strcmp`, `memcmp`) стандарт требует только знак результата, поэтому сравнивайте знаки.

## Покрытие кода: gcov

### Как связано с Check

Check отвечает на вопрос «**проходят ли тесты**». Gcov отвечает на вопрос «**какая часть кода вообще выполнялась тестами**». Это разные инструменты, но в связке они дают полную картину: зелёные тесты при покрытии в тридцать процентов почти ничего не доказывают.

Gcov это часть GCC. Он работает по двум видам служебных файлов.

### Как это работает

|Файл|Когда появляется|Что хранит|
|---|---|---|
|`.gcno`|при **компиляции** с флагами покрытия|структура программы: блоки, ветки, строки|
|`.gcda`|при **запуске** программы|счётчики: сколько раз выполнился каждый блок|

Затем `gcov` читает оба файла и показывает, какие строки и ветки были выполнены.

### Флаги компиляции

```bash
gcc --coverage -O0 -g -c s21_string.c -o s21_string.o
```

- `--coverage` включает `-fprofile-arcs -ftest-coverage` при компиляции и подключает нужную библиотеку при линковке;
- `-O0` отключает оптимизации: иначе компилятор перестраивает код и сопоставление со строками становится неточным;
- `-g` добавляет отладочную информацию.

Флаги покрытия нужны **только при сборке тестируемой библиотеки**. Если собрать с ними и сами тесты, то в отчёт попадёт код тестов, и процент искусственно вырастет.

Линковка тоже с `--coverage`:

```bash
gcc --coverage tests.o s21_string.o -o test $(pkg-config --libs check)
```

### Получение отчёта

Запустите тесты, после этого появятся `.gcda`:

```bash
./test
gcov s21_string.c
```

Результат это файл `s21_string.c.gcov`, где у каждой строки указан счётчик выполнений. Строки с `#####` ни разу не выполнялись.

Полезные ключи `gcov`:

|Ключ|Эффект|
|---|---|
|`-b`|добавляет статистику по ветвям (`if`, `?:`, циклы)|
|`-c`|абсолютные счётчики вместо процентов для ветвей|

### HTML-отчёт через lcov

Для наглядности используют `lcov` и `genhtml`:

```bash
lcov -t "s21_string" -o test.info -c -d .
genhtml -o report test.info
```

- `-c` собирает данные покрытия;
- `-d .` каталог с `.gcda` и `.gcno`;
- `genhtml` строит HTML-страницы, которые открываются в браузере (`report/index.html`).

Для учёта ветвей в новых версиях `lcov` добавляют `--rc branch_coverage=1`, в старых `--rc lcov_branch_coverage=1`.

### Типичные цели Makefile

```make
CC = gcc
CFLAGS = -Wall -Werror -Wextra -std=c11
CHECK_LIBS = $(shell pkg-config --cflags --libs check)

test: s21_string.a tests.o
	$(CC) tests.o s21_string.a -o test $(CHECK_LIBS)
	./test

gcov_report: clean
	$(CC) $(CFLAGS) --coverage -O0 -g -c s21_string.c -o s21_string.o
	$(CC) $(CFLAGS) -c tests.c -o tests.o
	$(CC) --coverage tests.o s21_string.o -o test $(CHECK_LIBS)
	./test
	lcov -t "s21_string" -o test.info -c -d .
	genhtml -o report test.info

clean:
	rm -rf *.o *.a *.gcda *.gcno *.gcov *.info report test
```

Обратите внимание на `clean` перед `gcov_report`: старые `.gcda` от другой сборки дают ошибки вроде «profiling: ... mismatch» или искажённые цифры.

### Частые проблемы с gcov

- **Нулевое покрытие или пустой отчёт.** Тесты не запускались, либо `.gcda` лежат в другом каталоге, либо библиотека собрана без `--coverage`.
- **Предупреждения о несоответствии версий.** Остались старые `.gcda` или `.gcno` после перекомпиляции. Решение: `make clean`.
- **Покрытие в режиме fork.** Как правило, работает без проблем: дочерние процессы Check завершаются штатно, и счётчики записываются. Если данные теряются, попробуйте `CK_FORK=no`.
- **Низкое покрытие веток при высоком покрытии строк.** Тесты не проверяют обе стороны условий. Добавляйте тесты на граничные случаи: пустая строка, `NULL`, нулевой размер, максимальные значения.
- **Покрытие тестов в отчёте.** Флаги `--coverage` попали в компиляцию тестов. Либо уберите их оттуда, либо отфильтруйте: `lcov --remove test.info '*tests*' -o test.info`.

### Что считать хорошим покрытием

Процент покрытия измеряет только факт выполнения строки, а не правильность проверки. Строка может быть «покрыта» вызовом без единого `ck_assert`. Ориентир: стремиться к высокому покрытию строк и ветвей, но главным критерием считать осмысленные проверки, особенно граничные случаи.

## Краткая шпаргалка

### Минимальный шаблон

```c
#include <check.h>
#include <stdlib.h>

START_TEST(test_name) {
  ck_assert_int_eq(1 + 1, 2);
}
END_TEST

Suite *my_suite(void) {
  Suite *s = suite_create("my");
  TCase *tc = tcase_create("core");
  tcase_add_test(tc, test_name);
  suite_add_tcase(s, tc);
  return s;
}

int main(void) {
  SRunner *sr = srunner_create(my_suite());
  srunner_run_all(sr, CK_NORMAL);
  int failed = srunner_ntests_failed(sr);
  srunner_free(sr);
  return failed == 0 ? EXIT_SUCCESS : EXIT_FAILURE;
}
```

### Что выбрать для проверки

|Что сравниваем|Макрос|
|---|---|
|`int`, `long`|`ck_assert_int_eq`|
|`size_t`, `unsigned`|`ck_assert_uint_eq`|
|строки|`ck_assert_str_eq`|
|буферы, массивы|`ck_assert_mem_eq`|
|указатели|`ck_assert_ptr_eq`|
|`double`, `float`|`ck_assert_double_eq_tol`|
|произвольное условие|`ck_assert` или `ck_assert_msg`|

### Команды для проверки качества

|Цель|Команда|
|---|---|
|запуск тестов|`./test`|
|отладка без fork|`CK_FORK=no gdb ./test`|
|утечки памяти|`CK_FORK=no valgrind --leak-check=full ./test`|
|покрытие по строкам|`gcov s21_string.c`|
|HTML-отчёт|`lcov -c -d . -o test.info && genhtml -o report test.info`|

Могу оформить этот конспект отдельным документом, чтобы его было удобно хранить и экспортировать, либо расширить какой-то раздел, например примерами тестов для конкретных функций `string.h` или `sprintf`.
