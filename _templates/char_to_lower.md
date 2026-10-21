<%*
const text = tp.file.selection();

if (!text || text.trim() === "") {
    new Notice("⚠️ Выделите текст");
    return;
}

// Убираем пробелы/переносы в конце и берём последний символ
const trimmed = text.replace(/\s+$/, "");
const lastChar = trimmed.slice(-1).toLowerCase();

tR += lastChar;
%>