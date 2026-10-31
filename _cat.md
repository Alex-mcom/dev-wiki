Описания в виде комментариев к прототипам, по файлам. Порядок совпадает с порядком вызовов сверху вниз.

**main.c**

```c
// Точка входа: разобрать аргументы, обработать файлы, вернуть код возврата.
int main(int argc, char* argv[]);
```

**options.h / options.c**

```c
// Публичная. Разбирает argv через getopt, накапливает флаги в options->flags,
// запоминает индекс первого файла в options->first_file_index.
// Неизвестная опция даёт kStatusError.
Status ParseOptions(int argc, char* argv[], Options* options);

// static. Превращает один символ опции в маску флагов.
// 'A' даёт v|E|T, 'e' даёт v|E, неизвестный символ даёт 0.
static unsigned FlagFromChar(int option_char);

// static. Приводит набор флагов к согласованному виду:
// если задан -b, сбрасывает -n (-b имеет приоритет).
static void NormalizeFlags(Options* options);
```

**cat.h / cat.c**

```c
// Публичная. Верхний уровень: инициализирует CatState, обходит список файлов
// (пустой список означает stdin), в конце сбрасывает буфер вывода.
// Возвращает kStatusError, если хоть один файл не открылся, не прочитался
// или не записался.
Status CatFiles(const Options* options, char* const files[], int count);

// static. Обрабатывает один путь: "-" это stdin, иначе fopen.
// Вызывает CatStream, закрывает файл (stdin не закрывает),
// при ошибке печатает сообщение в stderr и возвращает kStatusError,
// не прерывая работу над остальными файлами.
static Status CatPath(const char* path, CatState* state);

// static. Читает поток блоками по kBlockSize через fread до EOF и отдаёт
// каждый блок в ProcessBlock. По окончании проверяет ferror.
static Status CatStream(FILE* stream, CatState* state);

// static. Проходит по байтам блока и вызывает ProcessByte для каждого.
static void ProcessBlock(CatState* state, const unsigned char* block, size_t size);

// static. Ядро автомата для одного байта: решает, нужно ли его пропустить
// (-s), нумеровать ли строку, выводить ли байт, и обновляет состояние
// (at_line_start, prev_line_blank). Состояние переживает границы блоков и файлов.
static void ProcessByte(CatState* state, unsigned char byte);

// static. Если в начале строки нужна нумерация (-n для всех строк,
// -b только для непустых), увеличивает line_number и добавляет
// в вывод префикс "%6lu\t".
static void NumberLine(CatState* state, bool blank_line);

// static. Преобразует байт в видимое представление по флагам
// (\n даёт $\n при -E, \t даёт ^I при -T, управляющие символы и 127
// дают ^X и ^? при -v, байты >= 128 дают M- плюс то же) и добавляет результат
// в буфер вывода.
// Единственная точка вывода байта: выбирает представление по флагам // и добавляет его в буфер.
static void EmitByte(CatState* state, unsigned char byte);

// static. Копирует size байт в буфер вывода; если места не хватает,
// сначала сбрасывает буфер через FlushOutput.
static void AppendBytes(Output* out, const char* data, size_t size);

// static. Единственное место с fwrite: записывает накопленное содержимое
// буфера в stdout одним вызовом, обнуляет длину,
// при сбое записи ставит out->failed.
static void FlushOutput(Output* out);
```

Цепочка вызовов получается такой:

```
main → ParseOptions → (FlagFromChar, NormalizeFlags)
     → CatFiles → CatPath → CatStream → ProcessBlock → ProcessByte
                                                     → (NumberLine, EmitByte) → AppendBytes → FlushOutput
```

Все функции `cat.c`, кроме `CatFiles`, приватные, поэтому в `cat.h` попадает только она.

```c
// Единственная точка вывода байта: выбирает представление по флагам
// и добавляет его в буфер.
static void EmitByte(CatState* state, unsigned char byte) {
  char text[4];  // максимум "M-^X"
  size_t length = 0;

  if (byte == '\n') {
    if (state->flags & kShowEnds) {
      text[length++] = '$';
    }
    text[length++] = '\n';
  } else if (byte == '\t' && (state->flags & kShowTabs)) {
    text[length++] = '^';
    text[length++] = 'I';
  } else if ((state->flags & kShowNonprint) &&
             ((byte < 32 && byte != '\t') || byte >= 127)) {
    unsigned char value = byte;
    if (value >= 128) {
      text[length++] = 'M';
      text[length++] = '-';
      value -= 128;
    }
    if (value < 32 || value == 127) {
      text[length++] = '^';
      text[length++] = (value == 127) ? '?' : (char)(value + 64);
    } else {
      text[length++] = (char)value;
    }
  } else {
    text[length++] = (char)byte;  // обычный символ, а также \t без -T и любые байты без -v
  }
  AppendBytes(&state->out, text, length);
}
```

В итоге в `ProcessByte` для каждого непропущенного байта вызывается ровно одна `EmitByte`. Какие ветки выбираются:

- `\n`: просто перевод строки или `$\n` при `-E`;
- `\t`: `^I` при `-T`, иначе как есть (`-v` табуляцию не трогает);
- управляющие символы, `DEL` и байты `>= 128` при `-v`: `^X`, `^?`, `M-...`;
- всё остальное: символ без изменений.