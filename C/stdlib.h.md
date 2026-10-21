## rand()

```
int rand(void);
```

```c
#include <stdio.h>
#include <stdlib.h>
 
int main(void)
{
    int r_1 = rand();
    int r_2 = rand();
 
    printf("%d, %d, %d, %d, %d\n", r_1, r_2, rand(), rand(), rand());
 
    return 0;
}
```

Возвращает псевдослучайные целые значения в диапазоне от **0** до константы **RAND_MAX**. Значение `RAND_MAX` не меньше числа **32767**. Функция `rand()` и константа `RAND_MAX` определены в заголовочном файле `stdlib.h`

При каждом запуске программы функция `rand()` выдает абсолютно ту же самую последовательность значений псевдослучайных чисел. Это связано с тем, что алгоритм их генерации базируется на начальном состоянии регистров при запуске программы. А они каждый раз содержат одни и те же данные.

Значения псевдослучайных чисел распределены по равномерному закону в диапазоне [0; RAND_MAX]. То есть, с равной вероятностью может появиться любое значение из этого диапазона.

### srand()

```c
void srand(unsigned int seed);
```

Задает начальное значение «зерна» (**seed**) для датчика псевдослучайных чисел. Используется совместно с `rand()`

Чтобы изменения происходили каждый раз от запуска к запуску, необходимо при каждом запуске менять значение зерна. В практике программирования для этого часто используют еще одну функцию `time`, которая объявлена в заголовочном файле `time.h` и возвращает текущее время в секундах.

```c
srand(time(NULL));
```

### Генерация псевдослучайных чисел заданного диапазона (целые числа)

```c
int r = min + rand() % (max - min + 1);
```

```c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

int main(void) {
    srand(time(NULL));
    // [-10; 20]
    int min = -10;
    int max = 20;
    int r;

    for (int i = 0; i < 10; i++) {
        r = min + rand() % (max - min + 1);
        printf("%d ", r);
    }

    return 0;
}
```

Допустим, вам нужно сгенерировать число в диапазоне **от 10 до 20** (включая 10 и 20):
1. **`max - min + 1`** — вычисляет количество чисел в желаемом интервале: 20−10+1=11 элементов (числа: 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20).
2. **`rand() % 11`** — даёт остаток от деления, т.е. случайное число строго в диапазоне **от 0 до 10**.
3. **`10 + ...`** — сдвигает диапазон вправо на величину `min`, превращая [0,10] в [10,20].

### Вещественные

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    int min = -5, max = 5;
    double res = 0;

    for (int i = 0; i < 10; i++) {
        res = ((double)rand() / (double)RAND_MAX) * (max - min) + min;
        printf("%.2f", res);
        if (i < 9) printf(" ");
    }
    return 0;
}
```
## abs()

```c
int abs(int x);
```

Вычисляет модуль (абсолютное значение) целого числа.

```c
int val = -15;
int res = abs(val); // res = 15
```


## Преобразование из строк в числа

### `ato*`

Семейство функций `ato*` (ASCII to…) преобразует начальный фрагмент строки, содержащий число, в соответствующее числовое значение. Разбор прекращается на первом «нечисловом» символе. Если преобразование невозможно (нет цифр в начале строки) — возвращается **`0`**.

Названия расшифровываются как:
- `atoi` — **a**scii **to** **i**nteger;
- `atol` — **a**scii **to** **l**ong;
- `atoll` — **a**scii **to** **l**ong **l**ong (C99);
- `atof` — **a**scii **to** **f**loating point (double).

- **Не устанавливают `errno`**, не сигнализируют об ошибке.
- **Невозможно отличить** «результат 0» от «ошибки разбора» — обе ситуации возвращают `0`.
- Переполнение → **неопределённое поведение** (на практике — тихое усечение или мусор).

```c
#include <stdlib.h>

int    atoi (const char *str);
long   atol (const char *str);
long long atoll(const char *str);   // C99
double atof (const char *str);
```

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    printf("%d\n",   atoi("42"));          // 42
    printf("%d\n",   atoi("  -17abc"));    // -17
    printf("%d\n",   atoi("abc"));         // 0
    printf("%ld\n",  atol("2147483648"));  // 2147483648 (long)
    printf("%lld\n", atoll("9000000000")); // 9000000000
    printf("%f\n",   atof("3.14"));        // 3.140000
    printf("%f\n",   atof("2.5e3"));       // 2500.000000
    printf("%f\n",   atof("hello"));       // 0.000000
    return 0;
}
```

### `strto*`

Семейство `strto*` — это набор стандартных функций для **безопасного преобразования строк в числа**. Их главное отличие от «упрощённых» аналогов (`atoi`, `atof`, `atol`) — возможность **узнать, где закончился парсинг** и **обнаружить ошибки**.

Все функции семейства работают по единому принципу:
1. Пропускают ведущие пробельные символы.
2. Разбирают число в соответствии со своим форматом.
3. Возвращают значение и (опционально) записывают в `endptr` адрес первого символа, который **не является частью числа**.

| Функция     | Тип результата           | Формат числа                |
| ----------- | ------------------------ | --------------------------- |
| `strtol`    | `long int`               | целое, любое основание 2–36 |
| `strtoll`   | `long long int`          | целое, любое основание 2–36 |
| `strtoul`   | `unsigned long int`      | целое без знака             |
| `strtoull`  | `unsigned long long int` | целое без знака             |
| `strtof`    | `float`                  | с плавающей точкой          |
| `strtod`    | `double`                 | с плавающей точкой          |
| `strtold`   | `long double`            | с плавающей точкой          |

```c
long      strtol (const char *str, char **endptr, int base);
long long strtoll(const char *str, char **endptr, int base);
unsigned long      strtoul (const char *str, char **endptr, int base);
unsigned long long strtoull(const char *str, char **endptr, int base);

float       strtof (const char *str, char **endptr);
double      strtod (const char *str, char **endptr);
long double strtold(const char *str, char **endptr);
```


**Парсинг вещественных чисел из строки и занесение их в массив**

Данные на входе могут быть с пропусками чисел
```
csv:-0.3; 0.23;-23.0; 3.3;
csv:-0.3; 0.23;-23.0; 3.3;;54.2
```

```c
int get_data_csv(double nums[], int max_len, char str[]) {
    int count = 0;

    char* ptr = strchr(str, ':');
    if (ptr != NULL) {
        ptr++;

        while (*ptr && count < max_len) {
            char* end;
            double val = strtod(ptr, &end);

            if (ptr == end) break;

            nums[count++] = val;
            ptr = end;
            while (*ptr == ';' || *ptr == ' ' || *ptr == '\t') ptr++;
        }
    }

    return count;
}
```