<%*
const text = tp.file.selection();

if (!text || text.trim() === "") {
    new Notice("⚠️ Выделите текст");
    return;
}

// Берём последний символ (включая возможные пробелы/переносы в конце)
const trimmed = text.replace(/\s+$/, "");
const lastChar = trimmed.slice(-1).toUpperCase();

tR += lastChar;
%>