# Cyber Incident Analyst — n8n Workflow

מערכת אוטומציה ל-n8n שמנתחת לוגים ודוחות תקריות סייבר (TXT / PDF / DOCX), מסכמת אותם באמצעות Gemini API, מעשירה את הניתוח בלוגיקת ניתוב ודירוג סיכון דרך מיקרו-שירות Python, ומפיצה את התוצאה ל-Google Sheets, קובץ דוח, ו-Gmail (להתראות קריטיות).

## איך זה עובד

1. **קליטה** — `Schedule Trigger` בודק כל 30 שניות את `incoming_docs/` באמצעות `Read/Write Files from Disk`.
2. **זיהוי סוג קובץ** — `Switch` מנתב לפי סיומת (`txt` / `pdf` / `docx`) לאחד משלושה ענפי חילוץ טקסט מקבילים:
   - TXT / PDF → `Extract from File`
   - DOCX → הומר ל-base64 ונשלח ל-endpoint ייעודי במיקרו-שירות (`python-docx` לא נתמך כ-node מובנה)
   - שלושת הענפים מתאחדים ב-`Merge` לפורמט אחיד (`data`, `fileName`).
3. **ניתוח AI** — `Message a model` (Gemini, מודל `gemini-2.5-flash`) מקבל פרומפט קבוע שמחזיר JSON מבני: סיכום, חמירות (severity), סוג תקרית, ישויות מושפעות (IP/משתמש/שרת), IOCs, והמלצת פעולה.
4. **פירוק JSON** — `Code in JavaScript` מנקה markdown fences ומפרק את תשובת ה-AI לאובייקט עבודה.
5. **העשרת מטא-דאטה** — `HTTP Request` שולח לשירות Python (`/enrich`) שמחשב לוגיקה דטרמיניסטית (לא AI): `risk_score` (0–100), `routing_team` (צוות אחראי), `sla_minutes`, ו-`requires_escalation`.
6. **הפצת תוצאה** — שלושה ענפים מקבילים מ-`HTTP Request`:
   - `Append row in sheet` — שורה חדשה ב-Google Sheets לכל תקרית.
   - `Code in JavaScript1` → `Read/Write Files from Disk1` → `Execute Command` — בניית דוח `.md` קריא, כתיבתו ל-`output_docs/`, והעברת קובץ המקור ל-`incoming_docs/processed/` (כדי שלא יעובד פעמיים).
   - `If` (`requires_escalation == true`) → `Send a message` (Gmail) — התראה רק על תקריות שעברו סף סיכון.

## מיקרו-שירות Python (`app.py`)

Flask service עם שני endpoints:
- `POST /enrich` — מקבל ניתוח AI, מחזיר `risk_score`, `routing_team`, `sla_minutes`, `requires_escalation` לפי טבלאות lookup קבועות.
- `POST /extract-docx` — מקבל DOCX מקודד ב-base64, מחזיר טקסט גולמי (`python-docx`), כולל תוכן טבלאות.

רץ על המחשב המקומי (`localhost:5000`); n8n (בתוך Docker) פונה אליו דרך `host.docker.internal:5000`.

## הרצה

```powershell
# שירות Python
cd metadata-api
pip install flask python-docx
python app.py

# n8n (Docker)
docker run -d -it --name n8n `
  -p 5678:5678 `
  -e NODE_TLS_REJECT_UNAUTHORIZED=0 `
  -e NODES_EXCLUDE='[]' `
  -e N8N_RESTRICT_FILE_ACCESS_TO="" `
  -v n8n_data:/home/node/.n8n `
  -v <path>\incoming_docs:/data/incoming_docs `
  -v <path>\output_docs:/data/output_docs `
  docker.n8n.io/n8nio/n8n
```

ייבוא: ב-n8n → Import from File → `HW-project.json`.

## אתגרים שנפתרו במהלך הבנייה

| # | אתגר | פתרון |
|---|------|-------|
| 1 | `Local File Trigger` לא הופיע ברשימת ה-nodes (מושבת כברירת מחדל מטעמי אבטחה בגרסאות חדשות) | מעבר לארכיטקטורת polling: `Schedule Trigger` + `Read/Write Files from Disk` במקום file-watching |
| 2 | קבצים לא נראים בתוך הקונטיינר אף שנוצרו במחשב | נתיב `~/...` של Linux לא תקין ב-PowerShell בתוך `docker run -v`; תוקן לנתיב מוחלט של Windows (`C:\Users\...`) |
| 3 | `Access to the file is not allowed` | n8n מגביל גישת קבצים כברירת מחדל ל-`/home/node/.n8n-files`; נדרש `N8N_RESTRICT_FILE_ACCESS_TO` |
| 4 | ההגבלה נשארה גם אחרי הגדרת הנתיבים המורשים | תקלה בפענוח רשימת הנתיבים; נפתר בהגדרת `N8N_RESTRICT_FILE_ACCESS_TO=""` (ביטול ההגבלה לסביבת dev) |
| 5 | Gemini החזיר "No log content was provided" | שם השדה ב-`Extract from File` היה `data`, לא `text` כפי שהונח בפרומפט |
| 6 | `Cannot read properties of undefined ('ip_addresses')` | `Google Sheets` node מחזיר רק את העמודות שמופו בטבלה, לא את כל ה-JSON; תוקן בחיווט שני מסלולים מקבילים מ-`HTTP Request` (אחד ל-Sheets, אחד להמשך השרשרת) |
| 7 | תנאי ה-`If` (`requires_escalation`) תמיד נכשל אף שהערך היה `true` | השוואת סוגים (boolean מול string) ב-condition; תוקן בהגדרת סוג ההשוואה כ-Boolean |
| 8 | שגיאת OAuth ב-Gmail ("invalid or expired grant") | חידוש (reconnect) ה-credential דרך OAuth מחדש |
| 9 | DOCX לא נתמך ב-`Extract from File` | טיפול ייעודי: `Move File to Base64` ב-n8n + endpoint `/extract-docx` במיקרו-שירות Python עם `python-docx` |
| 10 | `Failed to parse DOCX: File is not a zip file` | קובץ הנעילה הזמני של Word (`~$filename.docx`) זוהה בטעות כ-DOCX תקין; נפתר בסגירת הקובץ ב-Word לפני ההרצה |
| 11 | `Cannot assign to read only property 'name'... Node 'Extract from File' hasn't been executed` | הפניה ישירה ל-node ספציפי (`$('Extract from File')`) שלא רלוונטי לכל ענפי הקבצים; תוקן בהפניה ל-`Merge` (הצומת המשותף לכל הסוגים) |
| 12 | `Service unavailable... high demand` מ-Gemini | מודל `gemini-2.0-flash` הוצא משימוש; מעבר ל-`gemini-2.5-flash` + הפעלת Retry on Fail (3 ניסיונות, 10 שניות בין ניסיונות) |

## מבנה תיקיות

```
n8n-cyber-analyst/
├── incoming_docs/        # קבצי קלט (txt/pdf/docx)
│   └── processed/        # קבצים שעובדו בהצלחה
├── output_docs/           # דוחות .md שנוצרו
└── metadata-api/
    └── app.py             # מיקרו-שירות Python
```
