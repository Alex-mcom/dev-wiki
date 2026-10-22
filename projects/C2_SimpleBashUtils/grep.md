**grep** (от англ. _Global Regular Expression Print_) — это консольная утилита в Unix/Linux, которая ищет в тексте строки, соответствующие заданному шаблону, и выводит их.

**grep** выводит **целиком** совпавшую строку, то есть если в строке совпала только часть, выведется вся строка всё равно:

```file1.txt
apple
Apple
applepen other text
banana brain
app
```

```bash
cat file.txt | grep "apple"

apple
applepen other text
```

Ищёт именно совпадения подстроки, а не совпадение всего слова:
```bash
cat tests/data/file1.txt | grep "ap"

apple
applepen other text
app and other text
```