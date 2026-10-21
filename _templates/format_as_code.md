<%*
// Берём выделенный текст
const text = tp.file.selection();

// Если ничего не выделено — предупреждаем и выходим
if (!text || text.trim() === "") {
    new Notice("⚠️ Выделите текст, который нужно отформатировать");
    return;
}

// Разбиваем на строки
const lines = text.split("\n");
const result = [];
let i = 0;

// Список поддерживаемых языков
const langRegex = /^(cpp|c\+\+|c|python|py|js|javascript|ts|typescript|java|kotlin|rust|go|sql|bash|sh|html|css|json|yaml|xml|php|ruby|swift|makefile)\s*$/i;

while (i < lines.length) {
    const line = lines[i];
    const match = line.match(langRegex);
    
    if (match) {
        let lang = match[1].toLowerCase();
        if (lang === "c++") lang = "cpp";
        if (lang === "py") lang = "python";
        if (lang === "js") lang = "javascript";
        if (lang === "ts") lang = "typescript";
        i++;
        
        // Собираем код до следующего маркера языка
        const codeLines = [];
        while (i < lines.length && !lines[i].match(langRegex)) {
            codeLines.push(lines[i]);
            i++;
        }
        
        // Убираем пустые строки по краям
        while (codeLines.length > 0 && codeLines[0].trim() === "") codeLines.shift();
        while (codeLines.length > 0 && codeLines[codeLines.length - 1].trim() === "") codeLines.pop();
        
        result.push("```" + lang);
        result.push(codeLines.join("\n"));
        result.push("```");
    } else {
        result.push(line);
        i++;
    }
}

// Возвращаем результат — Templater заменит выделение на него
tR += result.join("\n");
%>