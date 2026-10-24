В Си нет строкового типа данных, поэтому для хранения последовательности символов, как правило, используют массивы типа `char`

`char str[] = {'B', 'a', 'l', 'a', 'k', 'i', 'r', 'e', 'v', '\0'};`

![[Pasted image 20260915173330.png]]

Обратите внимание на последний символ `'\0'`. Он совпадает с числом `0`. Это маркер (метка) конца строки. Его часто называют **символом конца строки**. Каждая **корректная строка** в языке Си должна иметь такой символ. Иначе, строка считается некорректной. Поэтому длина строки всегда на один больше общего числа «видимых» символов. Маркер конца строки не отображается стандартными функциями работы со строками, а лишь учитывается ими для определения конца строки

>Каждая корректная строка в языке Си должна завершаться числом `0` (символ `'\0'`)

## Способы определения строк

```c
char str[] = {'B', 'a', 'l', 'a', 'k', 'i', 'r', 'e', 'v', '\0'};

char s[] = "Sergey Balakirev";

char buffer[512] = "Hello, World";

// "Hello, World"
char b[] = "Hel" "lo"          
		   ", World";
		   
char sp[] = "Hello, \ 
World";

char string[] = "a";   // строка из двух символов a0
```

## Представление строковых литералов в памяти

>Всё, что написано в двойных кавычках (`"..."`), хранится как последовательность байт (массив `char`) **в самом исполняемом файле**. При загрузке программы эти строки размещаются в **неизменяемой** области памяти (обычно сегмент .text). У каждого строкового литерала есть **уникальный адрес**.

При объявлении массива компилятор **копирует** символы строки (включая `'\0'`) в массив. При этом массив и строковый литерал находятся в **разных** областях памяти:

> [!Литерал]
> — в **неизменяемой** области (.text/.rodata)
> 
>Это позволяет не копировать одни и те же литералы в памяти, а просто ссылать на один и тот же адрес с этим литералом

> [!Массив]
>  — в **стековом фрейме** (локальная переменная) или в секции **.data** (если глобальный)

![[Pasted image 20260915200325.png|428]]

```c
int main(void) 
{
	char s[] = "Sergey Balakirev";
	char buffer[512] = "Hello, World";
	
	/* оба эти литерала хранятся в неизменяемой области памяти .text.
	При этом, когда происходит инициализация массива, компилятор копирует все
	эти литералы в стековый фрейм */
	
	return 0;
}
```

---
Если инициализация массива происходит в **глобальной области видимости** (вне функций), тогда все начальные данные для соответствующих перемененных сохраняются в секции `.data`, затем, в момент загрузки программы сразу выделяется память под массив и в эту область загружаются литералы:

```c
char buffer[512] = "Hello, World";
 
int main(void) 
{
	char s[] = "Sergey Balakirev";

	return 0;
}
```

![[Pasted image 20260915201419.png|437]]

---
**Указатели на строковые литералы**

```c
int main(void) 
{
	/* здесь мы ссылаемся на литерал, объявленный 
	в неизменяемой области памяти, следовательно его можно 
	ТОЛЬКО ЧИТАТЬ */
	const char* str = "Balakirev";
	
	/* а здесь уже происходит посимвольное копирование литерала в стековый 
	фрейм для массива s[], можно
	ЧИТАТЬ и ИЗМЕНЯТЬ */
	char s[] = "Balakirev";
	
	return 0;
}
```

>Указатели на строковые литералы рекомендуется объявлять с ключевым словом `const` следующим образом. Тогда уже на этапе компиляции будет выдана ошибка, при попытке изменения строки

```c
const char* str = "Balakirev";
```


## Экранирование и спецсимволы

Для экранирования используется символ: `\`

```c
char s[] = "I like programming in \"C\" language.";
```

| **escape-последовательность** | **Код в таблице** **ASCII** | **Описание**                                                        |
| ----------------------------- | --------------------------- | ------------------------------------------------------------------- |
| \n                            | 0x0A                        | перевод на новую строку                                             |
| \r                            | 0x0D                        | возврат каретки                                                     |
| \t                            | 0x09                        | символ табуляции                                                    |
| \'                            | 0x27                        | экранирование апострофа (одинарная кавычка)                         |
| \"                            | 0x22                        | экранирование двойной кавычки                                       |
| \?                            | 0x3F                        | экранирование вопросительного знака                                 |
| \\                            | 0x5C                        | экранирование обратного слеша                                       |
| \0                            | 0x00                        | символ конца строки (число 0)                                       |
| \x<число>                     |                             | символ с указанным шестнадцатеричным значением                      |
| \uxxxx                        |                             | символ с кодом xxxx в кодировке UNICODE (добавлено в стандарте C99) |

## ОПЕРАЦИИ НАД СТРОКАМИ

**Удаление символа из строки**
```c
char str[100] = "Best string!";

int i = 0;  // Индекс для чтения
int j = 0;  // Индекс для записи

while (str[i] != '\0') {
	if (str[i] != 's') {
		str[j] = str[i];
		j++;
	}
	i++;
}
str[j] = '\0';
```

**Удаление символов из строки по подстроке**
```c
#include <stdio.h>
#include <string.h>

void strip_str(char str[], const char del_symbols[]) {
    int i = 0;  // read
    int j = 0;  // write

    for (i = 0; str[i] != '\0'; i++) {
        if (strchr(del_symbols, str[i]) == NULL) {
            str[j++] = str[i];
        }
    }
    str[j] = '\0';
}

int main(void) {
    char str[100] = {0}, str2[20];
    fgets(str, sizeof(str) - 1, stdin);
    char* ptr_n = strrchr(str, '\n');
    if (ptr_n != NULL) *ptr_n = '\0';

    // здесь продолжайте функцию main
    strip_str(str, ".,-!?");
    puts(str);

    return 0;
}
```

**Подстановка подстроки в строку**
```c
char str[50] = "best string!";

char* add_str = "The ";
int insert_idx = 0;

int str_sz = 0;
while (str[str_sz] != '\0') str_sz++;

int add_str_sz = 0;
while (add_str[add_str_sz] != '\0') add_str_sz++;

for (int i = str_sz; i >= insert_idx; i--) {
	str[i + add_str_sz] = str[i];
}

for (int i = 0; i < add_str_sz; i++) {
	str[i + insert_idx] = add_str[i];
}
```

**Подсчёт слов в строке (разделитель - один или несколько пробелов)**
```c
#include <stdio.h>

void strip_string(char* str, int max_len) {
	/* Удаляет символ переноса строки, если он есть */
    int count = 0;
    while (*str++ != '\0' && count++ < max_len);

    if (count > 1) {
        str -= 2;
        if (*str == '\n') *str = '\0';
    }
}

int main(void) {
    char str[100];
    fgets(str, sizeof(str), stdin);
    strip_string(str, sizeof(str));

    int words_c = 0;
    int in_word = 0;

    for (int i = 0; str[i] != '\0'; i++) {
        if (str[i] == ' ') {
            // вышли из слова
            in_word = 0;
        } else if (in_word == 0) {
            // вошли в слово
            words_c++;
            in_word = 1;
        }
    }

    printf("%d", words_c);

    return 0;
}
```

**Поиск двух самых длинных строк в двумерном массиве строк**
```c
#include <stdio.h>
#include <string.h>

#define MAX_CITIES 6

int main(void) {
    char cities[MAX_CITIES][50 + 1];

    for (int i = 0; i < MAX_CITIES; i++) {
        scanf("%50s", cities[i]);
    }

    int idx1 = 0, idx2 = -1;
    size_t len1 = strlen(cities[idx1]), len2 = 0;

    for (int i = 1; i < MAX_CITIES; i++) {
        size_t cur = strlen(cities[i]);

        if (cur > len1) {
            idx2 = idx1;
            len2 = len1;
            idx1 = i;
            len1 = cur;
        } else if (cur > len2) {
            idx2 = i;
            len2 = cur;
        }
    }

    for (int i = 0; i < MAX_CITIES; i++) {
        if (i == idx1 || i == idx2) printf("%s ", cities[i]);
    }

    return 0;
}
```

**Собрать новую строку без латинских символов**
```c
#include <stdio.h>
#include <string.h>

int is_not_lat(char ch) {
    return (ch < 'A' || ch > 'Z') && (ch < 'a' || ch > 'z');
}

void copy_string(char* dst, size_t max_len_dst, const char* src,
                 int (*pred)(char)) {
    int i = 0;  // src
    int j = 0;  // dst
    while (j < max_len_dst - 1 && src[i] != '\0') {
        if (pred(src[i])) {
            dst[j++] = src[i];
        }
        i++;
    }
    dst[j] = '\0';
}

int main(void) {
    char str[100] = {0}, str2[20];
    fgets(str, sizeof(str) - 1, stdin);
    char* ptr_n = strrchr(str, '\n');
    if (ptr_n != NULL) *ptr_n = '\0';

    copy_string(str2, sizeof(str2), str, is_not_lat);
    puts(str2);

    return 0;
}
```

**Подсчёт вхождений подстроки в строке**
```c
#include <stdio.h>
#include <string.h>

int main(void) {
    char str[100] = {0};
    fgets(str, sizeof(str) - 1, stdin);
    char* ptr_n = strrchr(str, '\n');
    if (ptr_n != NULL) *ptr_n = '\0';

    // здесь продолжайте программу
    char* ptr = str;
    int counter = 0;
    while ((ptr = strstr(ptr, "is")) != NULL) {
        counter++;
        ptr += 2;
    }

    printf("%d", counter);

    return 0;
}
```

### Ввод

**Обработка ввода, когда мы вводим значения с новой строки (не учитывает пустые строки)**

```
Барнаул
Кишинев
Ярославль
Николаев

Уфа
Москва
Санкт-Петербург

Самара
Воронеж


Смоленск
```

Создаст двумерный массив в памяти вида: 
![[Снимок экрана 2026-09-19 в 14.03.03.png|270]]

```c
#include <stdio.h>
#include <string.h>

#define MAX_ITEMS 10
#define ITEM_SIZE 51

int main(void) {
    char cities[MAX_ITEMS][ITEM_SIZE] = {0};

    int count = 0;
    while (count < MAX_ITEMS &&
           fgets(cities[count], ITEM_SIZE, stdin) != NULL) {
        char* ptr = strrchr(cities[count], '\n');
        if (ptr != NULL) {
            *ptr = '\0';
        }

        if (strlen(cities[count]) != 0) count++;
    }
    return 0;
}
```

**Ввод значений через пробел (каждое слово - отдельный элемент**
```c
#include <stdio.h>

#define MAX_ITEMS     100
#define MAX_ITEM_SIZE 64
#define STR_(x) #x
#define STR(x)  STR_(x)

int main(void) {
    char words[MAX_ITEMS][MAX_ITEM_SIZE] = {{0}};
    int count = 0;
    while (count < MAX_ITEMS &&
           scanf("%" STR(MAX_ITEM_SIZE - 1) "s", words[count]) == 1) {
        count++;
    }
    
    ...
```

### Замена

**Замена подряд идущих пробелов на одинарные**
```c
char str[50] = "  the Best   language  C  ";
// "the Best language C"

int i = 0;  // read
int j = 0;  // write

while (str[i]) {
	if (str[i] != ' ') {
		str[j++] = str[i++];
	} else {
		while (str[i] == ' ') i++;
		str[j++] = ' ';
	}
}

str[j] = '\0';
```

**Замена всех вхождений '-' на строку "-+-"**
```c
#include <stdio.h>
#include <string.h>

int main(void) {
    char str[100] = {0};
    fgets(str, sizeof(str) - 1, stdin);
    char* ptr_n = strrchr(str, '\n');
    if (ptr_n != NULL) *ptr_n = '\0';

    // здесь продолжайте программу
    int i = 0;
    int len = strlen(str);
    while (str[i] != '\0') {
        if (str[i] == '-') {
            len += 2;
            
            for (int k = len - 1; k > i + 2; k--) {
                str[k] = str[k - 2];
            }
            str[len] = '\0';

            str[++i] = '+';
            str[++i] = '-';
        }
        i++;
    }

    puts(str);

    return 0;
}
```

### Парсинг

**Парсит все вещественные значения из строки `str` в массив**

```
csv: 0.5; 24.35; -45.36; 100.0;65.3; -44.3
```

```c
#include <ctype.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(void) {
    char str[200] = {0};
    fgets(str, sizeof(str) - 1, stdin);
    char* ptr_n = strrchr(str, '\n');
    if (ptr_n != NULL) *ptr_n = '\0';

    double res[50] = {0.0};

    char* ptr = strchr(str, ':');

    int count = 0;
    if (ptr != NULL) {
        while (*ptr) {
            if (isdigit((unsigned char)*ptr) || *ptr == '-') {
                res[count++] = atof(ptr);
                while (*ptr && *ptr != ';') ptr++;
            } else {
                ptr++;
            }
        }
    }

    return 0;
}
```

### Поиск

**Поиск подстроки в строке без учёта регистра**
Регистр строки, в которой ведётся поиск, не учитывается. Регистр строки, которую ищем, учитывается
```c
int find_no_case(const char* str, const char* search_str) {
    for (int i = 0; str[i]; i++) {
        if (tolower(str[i]) == search_str[0]) {
            int j = 1;
            while (str[i + j] && search_str[j] &&
                   tolower(str[i + j]) == search_str[j])
                j++;
            if (search_str[j] == '\0') return 1;  // дошли до конца паттерна
        }
    }

    return 0;
}
```

## Библиотека `string.h`

```
strchr      - поиск 1 символа ->                                [char]  
strrchr     - поиск 1 символа <-                                [right char]  
strstr      - поиск строкового литерала(подстроки) ->           [string]  
strbprk     - поиск хотя бы 1 символа из подстроки ->           [pointer break]  
strcmp      - сравнение строк на равенство                      [comparison]  
strncmp     - сравнение первых n элементов на равенство ->      [comparison]  
strlen      - длина строки                                      [length]  
strcpy      - копирование 2-ой строки в первую                  [copy]  
strncpy     - копирование 2-ой строки в первую с указанием лимита копирования  
strcat      - добавление  2-ой строки к первой                  [concatenate]  
strncat     - добавление  2-ой строки к первой с указанием лимита добавления 
```

### `strlen()`

`size_t strlen(const char* buf)`

Возвращает длину строки **в БАЙТАХ** до ноль-терминатора (не включительно)

```c
char str[100] = "hello world!";
size_t len = strlen(str); // 12
```

Если **латиница**: 

```c
Ivanov Petrov Sidorov

[6] => [Ivanov]
[6] => [Petrov]
[7] => [Sidorov]
```

Но, а если **кириллица**, то **strlen()** выдаёт **двойной размер**

```c
Иванов Петров Сидоров

[12] => [Иванов]
[12] => [Петров]
[14] => [Сидоров]
```

### Копирование
#### `strcpy()`

`char* strcpy(char* dest, const char* src)`

Копирует в **dest** все символы строки из src, **пока** не встретится символ конца строки (он тоже копируется). Возвращает указатель на **dest**. Не учитывает выход за границы **dest**. Рекомендуется применять, только когда `dest >= source`

```c
char dest[10];

strcpy(dest, "hello");
// dest = "hello"
```

#### `strncpy()`

`char* strncpy(char* dest, const char* src, size_t max_len)`

Работает также, как `strcpy()`, но дополнительно еще проверяет, чтобы максимальное количество копируемых символов не превышало значения **max_len**. Такой подход считается более безопасным.

>**`strncpy` не гарантирует `'\0'`-терминатор**, если `strlen(src) >= n`. Именно поэтому в примере вручную пишут `d_1[max_len] = '\0';`

```c
char s_1[100] = "Source string";
char d_1[10], d_2[20];

int max_len = sizeof(d_1) - 1;
strncpy(d_1, s_1, max_len);

d_1[max_len] = '\0';
// d_1 = "Source st"

strncpy(d_2, "Balakirev", 5);
d_2[5] = '\0';
// d_2 = "Balak"
```

### Конкатенация
#### `strcat()`

`char* strcat(char* dest, const char* src)`

Добавляет все символы строки **src** к концу строки **dest** (включая символ конца строки). Не учитывает выход за границы **dest**. Рекомендуется применять, только когда `dest >= source`

- Возвращает **указатель на строку назначения** (`dest`).

```c
char dest[10] = "I'm";
strcat(dest, " human"); // "I'm human"
```

#### `strncat()`

`char* strncat(char* dest, const char* src, size_t max_add)`

Работает также, как и `strcat()`, но добавляет не более **max_add** символов в строку. Записывает последним символом `'\0'`

```c
char dest[10] = "I'm";

size_t add_max = sizeof(dest) - strlen(dest) - 1;
strncat(dest, " computer", add_max); // dest = "I'm compu"
```

### Сравнение

Строки считаются равными, если равны их **длины** и все соответствующие **символы**
![[Pasted image 20260918211536.png]]

```c
// Например, следующие две строки считаются равными:

char s1[12] = "Hello";
char s2[10] = "Hello";
```

#### `strcmp()`

`int strcmp(const char* str1, const char* str2);`

Выполняет **лексикографическое сравнение строк** по кодам из кодовой таблицы, а не по алфавиту. Возвращает `0`, если строки **равны**. При неравенстве строк возвращает **ненулевое целое число**:

- **< 0** — если первая строка лексикографически **меньше** второй;
- **> 0** — если первая строка лексикографически **больше** второй;
- **0** — если строки равны.

```c
strcmp("abc", "abd")  // < 0
strcmp("abd", "abc")  // > 0
strcmp("abc", "abc")  // = 0
```

#### `strncmp()`

`int strncmp(const char* str1, const char* str2, size_t max_len);`

Работает также, как и `strcmp()`, но позволяет задавать сравнение только первых **max_len** символов в обеих строках, а остальные - игнорировать.

```c
strncmp("ab9999", "ab1111", 2); // = 0
```

### Поиск символов и подстрок

#### `strchr()`

`char* strchr(const char* str, int val);`

Выполняет поиск символа **слева-направо** с кодом **val** в строке **str**. Возвращает указатель на ячейку памяти, где был первым найден символ, либо значение **NULL**, если поиск не увенчался успехом.

```c
char s[] = "string";
char* ptr_ch = strchr(s, 'r');

printf("Начало строки: %p\n", s); // 0x16b836da4
printf("Адрес символа \"r\": %p\n", ptr_ch); // 0x16b836da6
printf("Индекс символа: %ld", ptr_ch - s); // 2
```

#### `strrchr()`

`char* strrchr(const char* str, int val);`

Выполняет поиск символа **справа-налево** с кодом **val** в строке **str**. Возвращает указатель на ячейку памяти, где был первым найден символ, либо значение **NULL**, если поиск не увенчался успехом.

```c
/* Читаем строку и удаляем перенос строки, если он есть */

char str[100] = {0};

fgets(str, sizeof(str), stdin);
char* ptr = strrchr(str, '\n');
if (ptr != NULL) *ptr = '\0';
```
#### `strstr()`

`char* strstr(const char* str, const char* find);`

Выполняет поиск **слева-направо** подстроки **find** в строке **str**. Возвращает указатель на первый найденный фрагмент подстроки, либо значение **NULL**, если поиск не увенчался успехом.

```c
char* ptr = strstr("hello", "ll"); // ptr = "llo"
char* ptr2 = strstr("hello", "zero"); // ptr = NULL
```

#### `strpbrk()`

`char* strpbrk(const char* str, const char* find);`

Выполняет поиск **слева-направо** любого символа из набора символов (строки) **find** в строке **str**. Возвращает указатель на первый символ из `str`, который присутствует в `find`, либо `NULL`, если такого символа нет.

```c
char pass[] = "dfdfg90!#$$A";
char *ptr = strpbrk(pass, "@!#$^&?");

printf("pass = %p\nptr = %p\n", pass, ptr);
if(ptr != NULL)
   printf("ptr: %s\n", ptr);
```

#### `memchr()` и `memrchr()` - поиск символа в памяти

```c
void *memchr(const void *s, int c, size_t n);

void *memrchr(const void *s, int c, size_t n);
```

Функция **memchr()** ищет символ `c` в первых `n` байтах той области памяти, на которую указывает `s`. Совпадение первого байта с `c` (представленным как `unsigned char`) останавливает выполнение операции.

Функция **memrchr()** подобна функции **memchr**, за исключением того, что поиск ведется в обратном направлении, вместо начала, с конца `n` байт, на которые указывает `s`. 

- `s` — откуда начинать поиск
- `c` — что искать
- `n` — **сколько байт** смотреть вперёд

**Возвращаемые значения:**

Функции **memchr()** и **memrchr()** возвращают указатель на совпадающий байт или `NULL`, если символ не найден в данной области памяти.