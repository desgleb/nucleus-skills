---
name: docx
description: "Work with .docx Word documents: read content, edit text, add comments, change metadata, validate, accept tracked changes, and save — all while preserving original formatting. Use this skill whenever you need to read a .docx file, extract text from a Word document, modify document content (find & replace), add review comments, change document properties (author, title), create modified copies, or work with tracked changes. Triggers on any task involving .docx, Word documents, договоры, ТЗ, КП, акты, or any document editing task — even if the user doesn't explicitly say 'docx'."
---

# DocxSkill

Work with `.docx` (and `.pptx`) files by unpacking them into XML, making precise edits, and repacking — preserving all original formatting, styles, and structure. This approach avoids the formatting loss that libraries like python-docx can cause on save.

## Configuration

**Default Author**: Глеб Дешин | Мерусофт
**Python**: `~/.claude/venvs/docx/bin/python3` (defusedxml, lxml)
**Scripts**: `~/.claude/skills/docx/scripts/`
**Workspace**: `.agent/skills/docx_skill/workspace/`
**LibreOffice**: `soffice` (for accept_changes)

IMPORTANT: Always run scripts from the scripts directory:
```bash
cd ~/.claude/skills/docx/scripts && ~/.claude/venvs/docx/bin/python3 <script.py> ...
```

## Standard Workflow

1. **Setup Workspace**:
    * Working Directory: `.agent/skills/docx_skill/workspace/<session_id>`.
    * Source File: путь от пользователя или скачанный с Яндекс Диска.

2. **Unpack**:
    ```bash
    cd ~/.claude/skills/docx/scripts && ~/.claude/venvs/docx/bin/python3 unpack.py "/path/to/file.docx" "/abs/path/workspace/temp_doc"
    ```
    Автоматически: распаковка ZIP, pretty-print XML, merge adjacent runs, simplify tracked changes.
    Опции: `--merge-runs false`, `--simplify-redlines false`.

3. **Read & Analyze**:
    ```bash
    cd ~/.claude/skills/docx/scripts && ~/.claude/venvs/docx/bin/python3 extract_text.py "/abs/path/workspace/temp_doc"
    ```
    Выводит текст с номерами параграфов `[N]`, заголовки как `# / ## / ###`, таблицы в markdown-формате.

4. **Edit** (один или несколько вариантов):
    * **Find & Replace** (сохраняет форматирование, работает с текстом, разбитым на несколько run):
      ```bash
      cd ~/.claude/skills/docx/scripts && ~/.claude/venvs/docx/bin/python3 find_replace.py "/abs/path/workspace/temp_doc" "старый текст" "новый текст"
      # Или заменить все вхождения:
      cd ~/.claude/skills/docx/scripts && ~/.claude/venvs/docx/bin/python3 find_replace.py "/abs/path/workspace/temp_doc" "старый" "новый" --all
      ```
    * **Add Comment** (4-файловая система: comments, commentsExtended, commentsIds, commentsExtensible):
      ```bash
      cd ~/.claude/skills/docx/scripts && ~/.claude/venvs/docx/bin/python3 comment.py "/abs/path/workspace/temp_doc" 0 "Comment text" --author "Глеб Дешин | Мерусофт" --initials "ГД"
      # Reply to comment 0:
      cd ~/.claude/skills/docx/scripts && ~/.claude/venvs/docx/bin/python3 comment.py "/abs/path/workspace/temp_doc" 1 "Reply text" --parent 0
      ```
      After running, manually add markers to `document.xml` as instructed in the output.
      Markers must be direct children of `<w:p>`, never inside `<w:r>`.
    * **Direct XML Editing**: редактировать `word/document.xml` напрямую для изменения форматирования:
      - Цвет текста: `<w:color w:val="FF0000"/>` внутри `<w:rPr>`
      - Жирный: `<w:b/>` внутри `<w:rPr>`
      - Курсив: `<w:i/>` внутри `<w:rPr>`
    * **Metadata** (автор, заголовок):
      ```bash
      # Прочитать:
      cd ~/.claude/skills/docx/scripts && ~/.claude/venvs/docx/bin/python3 metadata.py "/abs/path/workspace/temp_doc"
      # Обновить:
      cd ~/.claude/skills/docx/scripts && ~/.claude/venvs/docx/bin/python3 metadata.py "/abs/path/workspace/temp_doc" creator="Глеб Дешин" title="Новый заголовок"
      ```

5. **Pack & Save** (with validation):
    ```bash
    timestamp=$(date +%Y-%m-%d-%H-%M)
    cd ~/.claude/skills/docx/scripts && ~/.claude/venvs/docx/bin/python3 pack.py "/abs/path/workspace/temp_doc" "/path/to/output/${timestamp}__file.docx" --original "/path/to/original.docx"
    ```
    Автоматически: XSD-валидация, auto-repair (paraId/durableId), tracked changes validation, XML condensing.
    Опции: `--validate false` (пропустить валидацию), `--original` (для сравнения с оригиналом).

## Additional Tools

### Validate (`office/validate.py`)
Standalone валидация без упаковки:
```bash
cd ~/.claude/skills/docx/scripts && ~/.claude/venvs/docx/bin/python3 -m office.validate "/abs/path/workspace/temp_doc" --original "/path/to/original.docx" --auto-repair -v
```
Проверяет: XSD schemas, namespaces, unique IDs, file references, content types, whitespace preservation, tracked changes consistency, comment markers.

### Accept Tracked Changes (`accept_changes.py`)
Принять все tracked changes через LibreOffice:
```bash
cd ~/.claude/skills/docx/scripts && ~/.claude/venvs/docx/bin/python3 accept_changes.py "/path/to/input.docx" "/path/to/output_clean.docx"
```

### Create DOCX from scratch (`docx-js`)
Создание нового документа через JavaScript:
```bash
NODE_PATH=$(npm root -g) node -e "
const docx = require('docx');
const fs = require('fs');
const doc = new docx.Document({
  sections: [{
    children: [
      new docx.Paragraph({ children: [new docx.TextRun('Hello')] })
    ]
  }]
});
docx.Packer.toBuffer(doc).then(b => fs.writeFileSync('output.docx', b));
"
```

## Tools Reference

| Script | Назначение |
|---|---|
| `unpack.py` | Распаковка DOCX/PPTX/XLSX + merge runs + simplify redlines |
| `extract_text.py` | Текст с нумерацией параграфов, заголовки, таблицы |
| `find_replace.py` | Поиск/замена с сохранением форматирования (cross-run) |
| `comment.py` | Комментарии (4 XML файла) + replies |
| `metadata.py` | Чтение/редактирование core properties |
| `pack.py` | Упаковка + валидация + auto-repair |
| `accept_changes.py` | Принятие tracked changes (LibreOffice) |
| `office/validate.py` | Standalone XSD валидация |
| `office/soffice.py` | LibreOffice wrapper (headless, auto-shim) |

## Tracked Changes (Redlining)

При работе с документами, содержащими tracked changes:

- **Удаление текста**: обернуть в `<w:del w:id="..." w:author="Claude" w:date="...">` с `<w:delText>` вместо `<w:t>`
- **Вставка текста**: обернуть в `<w:ins w:id="..." w:author="Claude" w:date="...">`
- **Отклонить чужую вставку**: вложить `<w:del>` внутрь чужого `<w:ins>`
- **Восстановить чужое удаление**: добавить `<w:ins>` после чужого `<w:del>`
- Валидатор проверит, что все изменения корректно отмечены

## Tips

- Номера параграфов из `extract_text.py` помогают быстро найти нужное место в `document.xml`
- Для поиска XML-фрагмента: `grep -nC 3 "текст" .../word/document.xml`
- Таблицы: `<w:tbl>` → `<w:tr>` (строка) → `<w:tc>` (ячейка) → `<w:p>` (параграф)
- `xml:space="preserve"` обязателен на `<w:t>` с пробелами по краям
- Comment markers (`commentRangeStart/End`) — прямые дети `<w:p>`, никогда внутри `<w:r>`
