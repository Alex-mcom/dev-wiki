## Модуль числа
### fabs

```c
double fabs(double x)
```

Вычисляет модуль **вещественного** числа.

```c
double val = -5.25;
double res = fabs(val); // res = 5.25
```

## Округление чисел

![[Снимок экрана 2025-01-06 в 01.45.06.avif]]
### round()

```c
double round(double x)
```

Округляет вещественное число до ближайшего целого по математическим правилам (0.5 округляется в большую по модулю сторону).

```c
double r1 = round(2.3);  // 2.0
double r2 = round(2.5);  // 3.0
double r3 = round(-2.5); // -3.0
```

### floor()

```c
double floor(double x)
```

Округляет вещественное число вниз до ближайшего меньшего или равного целого (округление к −∞).

```c
double r1 = floor(2.8);  // 2.0
double r2 = floor(-2.1); // -3.0
```

### ceil()

```c
double ceil(double x)
```

Округляет вещественное число вверх до ближайшего большего или равного целого (округление к +∞).

```c
double r1 = ceil(2.1);  // 3.0
double r2 = ceil(-2.8); // -2.0
```

### trunc()

```c
double trunc(double x)
```

Отбрасывает дробную часть вещественного числа (округление в сторону нуля).

```c
double r1 = trunc(2.9);  // 2.0
double r2 = trunc(-2.9); // -2.0
```

## Степени и корни

### sqrt()

```c
double sqrt(double x)
```

Вычисляет квадратный корень

```c
double res = sqrt(16.0); // res = 4.0
```

### cbrt()

```c
double cbrt(double x)
```

Вычисляет кубический корень. Корректно работает и с отрицательными числами.

```c
double res = cbrt(-27.0); // res = -3.0
```

### pow()

```c
double pow(double x, double y)
```

Возводит число **x** в степень **y**.

```c
double res = pow(2.0, 3.0); // res = 8.0 (2 в кубе)
```

### exp()

```c
double exp(double x)
```

Вычисляет значение экспоненциальной функции ex.

```c
double res = exp(1.0); // res ≈ 2.718282 (число e)
```

## Логарифмы

### `double log(double x)`

### `double log2(double x)`

### `double log10(double x)`

## Тригонометрия

> **Примечание:** Перевод градусов в радианы выполняется по формуле: rad=deg⋅180.0π​.

### `double sin(double x)`

### `double cos(double x)`

### `double tan(double x)`

### `double asin(double x)`

### `double acos(double x)`

### `double atan(double x)`