# הבודקת — אפיון מערכת לניהול ואימות שאלות טריוויה

מערכת ווב לניהול מחסן שאלות לשעשועון: ייבוא מגוגל שיט, אימות אוטומטי של התשובה
הנכונה מול שני מקורות מוסמכים, בניית "תיק תשובה", אישור ידני ע"י בודק, ושיבוץ
למתמודדים.

- שפת ממשק: עברית, RTL מלא.
- מזהים בקוד (טבלאות, שדות, enum) באנגלית. תוכן למשתמש בעברית.
- סטאק יעד: Lovable → React + Vite + TypeScript + Tailwind + shadcn/ui,
  Supabase (Postgres + Auth + RLS + Edge Functions), Lovable AI Gateway לאימות.

---

## 1. מודל הנתונים

### 1.1 מבנה סיבובים

| סיבוב | מספר תשובות |
|---|---|
| 1, 2, 3 | 4 |
| 4, 5, 6 | 3 |
| 7 | 2 |

מספר התשובות קובע לאיזו **קבוצת סיבובים** השאלה כשירה. הסיבוב המדויק נקבע ע"י
העורך/מנהל. אילוץ DB: `round_number` מול `option_count` חייבים להתאים לטבלה.

### 1.2 טבלאות

```
profiles
  id            uuid PK → auth.users(id) ON DELETE CASCADE
  full_name     text not null
  email         text
  created_at    timestamptz default now()

app_role        enum ('admin', 'checker', 'writer')

user_roles                       -- לעולם לא עמודת role על profiles (אבטחה)
  id            uuid PK
  user_id       uuid → profiles(id) ON DELETE CASCADE
  role          app_role not null
  unique (user_id, role)

writers                          -- מיפוי שם הכותב מהשיט למשתמש אמיתי
  id            uuid PK
  display_name  text not null unique   -- לדוגמה: "גרנות"
  user_id       uuid → profiles(id) null
  active        boolean default true

import_batches
  id            uuid PK
  source        text not null          -- 'google_sheet' | 'csv_upload' | 'manual'
  source_ref    text                   -- URL של השיט / שם קובץ
  imported_by   uuid → profiles(id)
  row_count     int
  created_at    timestamptz default now()

question_status enum (
  'draft',                 -- יובאה, לא נבדקה
  'writer_reviewed',       -- עברה בדיקה ראשונה של הכותב
  'verifying',             -- בבדיקה אוטומטית (רץ עכשיו)
  'verified',              -- אושרה אוטומטית, תיק תשובה מוכן
  'needs_rewrite',         -- ניסוח לא ברור / יותר מתשובה נכונה אחת
  'insufficient_sources',  -- לא נמצאו 2 מקורות מוסמכים
  'rejected',              -- התשובה של הכותב שגויה/לא מדויקת
  'checker_approved',      -- בודק אישר ידנית
  'assigned',              -- שובצה למתמודד
  'used'                   -- נשאלה בשידור
)

questions
  id                  uuid PK
  writer_id           uuid → writers(id) not null
  category            text not null
  text                text not null
  option_count        int not null check (option_count in (2,3,4))
  correct_index       int not null       -- 1-based, כמו בשיט
  round_number        int null check (round_number between 1 and 7)
  difficulty          int null check (difficulty between 1 and 5)   -- 1 קל … 5 קשה מאוד
  writer_notes        text               -- עמודת "הערות" מהשיט
  status              question_status not null default 'draft'
  rejection_reason    text               -- מדוע נפסלה (מוצג לכותב)
  suggested_text      text               -- ניסוח מדויק יותר שהמערכת הציעה
  rewrite_decision    text               -- 'accepted' | 'rethink' | null
  dossier_summary     text               -- "כמה מילים על התשובה הנכונה"
  latest_verification_id uuid → verifications(id) null
  import_batch_id     uuid → import_batches(id) null
  source_row          int                -- מספר השורה בשיט, לאיתור חזרה
  created_by          uuid → profiles(id)
  created_at          timestamptz default now()
  updated_at          timestamptz default now()
  check (correct_index <= option_count)
  check (round_number is null or
         (round_number between 1 and 3 and option_count = 4) or
         (round_number between 4 and 6 and option_count = 3) or
         (round_number = 7 and option_count = 2))

question_options
  id            uuid PK
  question_id   uuid → questions(id) ON DELETE CASCADE
  position      int not null       -- 1-based
  text          text not null
  is_correct    boolean not null default false
  one_liner     text               -- "שורה אחת" על תשובה לא נכונה
  unique (question_id, position)

verdict enum ('approved','rejected','needs_rewrite','insufficient_sources')

verifications                      -- היסטוריה: ריצה חדשה לא מוחקת קודמת
  id                  uuid PK
  question_id         uuid → questions(id) ON DELETE CASCADE
  verdict             verdict not null
  confidence          numeric(3,2)      -- 0.00–1.00
  model               text
  writer_choice_correct boolean
  found_correct_index int null           -- מה המערכת מצאה כנכון
  multiple_correct    boolean default false
  reasoning           text               -- בעברית, מוצג לבודק
  clarity_issues      jsonb default '[]' -- מערך מחרוזות
  suggested_text      text
  difficulty_suggestion int
  raw_response        jsonb              -- לדיבאג
  question_snapshot   jsonb              -- נוסח השאלה והתשובות בזמן הבדיקה
  created_at          timestamptz default now()

verification_sources
  id              uuid PK
  verification_id uuid → verifications(id) ON DELETE CASCADE
  url             text not null
  domain          text not null          -- דומיין רשום, לבדיקת אי-תלות
  publisher       text
  title           text
  tier            text                   -- ראה §3
  quote           text                   -- ציטוט רלוונטי (בעברית או מקור)
  supports_index  int                    -- איזו תשובה המקור תומך בה
  checked_ok      boolean default true   -- עבר סינון דומיינים

checker_approvals
  id            uuid PK
  question_id   uuid → questions(id) ON DELETE CASCADE
  checker_id    uuid → profiles(id)
  note          text
  created_at    timestamptz default now()
  unique (question_id, checker_id)

contestants
  id            uuid PK
  name          text not null
  notes         text
  active        boolean default true
  created_at    timestamptz default now()

assignments                        -- שיבוץ שאלה למתמודד
  id                uuid PK
  question_id       uuid → questions(id) ON DELETE CASCADE
  contestant_id     uuid → contestants(id)
  round_number      int
  assigned_by       uuid → profiles(id)
  is_active         boolean default true
  transferred_from  uuid → assignments(id) null
  transfer_reason   text
  created_at        timestamptz default now()
  -- שיבוץ פעיל אחד לכל שאלה:
  -- create unique index on assignments(question_id) where is_active;

audit_log
  id            uuid PK
  actor_id      uuid → profiles(id)
  entity        text          -- 'question' | 'assignment' | 'verification' | ...
  entity_id     uuid
  action        text          -- 'status_change' | 'assign' | 'transfer' | 'approve' | ...
  before        jsonb
  after         jsonb
  created_at    timestamptz default now()
```

### 1.3 כלל השיבוץ — קריטי

> אחרי שיבוץ שאלה למתמודד **לא ניתן לבטל** את השיבוץ, אלא רק **להעביר** למתמודד
> אחר.

מימוש: `DELETE` על `assignments` חסום ברמת RLS. העברה = טרנזקציה אחת
(`transfer_assignment` RPC): השורה הקיימת → `is_active = false`, ונוצרת שורה
חדשה עם `transferred_from` = המזהה הקודם + `transfer_reason` חובה. כך שרשרת
ההעברות שמורה במלואה, וה-UI מציג אותה בטיימליין של השאלה.

---

## 2. מכונת מצבים

```
draft
  └─(הכותב מבצע בדיקה ראשונה)→ writer_reviewed
       └─(הפעלת בדיקה אוטומטית)→ verifying
            ├─ verdict=approved             → verified
            ├─ verdict=rejected             → rejected   (+rejection_reason)
            ├─ verdict=needs_rewrite        → needs_rewrite (+suggested_text)
            └─ verdict=insufficient_sources → insufficient_sources

needs_rewrite
  ├─(הכותב בוחר את הניסוח שהמערכת הציעה)→ writer_reviewed  [rewrite_decision='accepted']
  └─(הכותב מסמן "לחשיבה על ניסוח מחדש")→ draft            [rewrite_decision='rethink']

rejected / insufficient_sources
  └─(הכותב מתקן תשובה/שאלה)→ draft          -- הפסילה נשמרת בהיסטוריה

verified
  └─(בודק מאשר ידנית)→ checker_approved
       └─(עורך משבץ למתמודד)→ assigned
            └─(העברה למתמודד אחר)→ assigned   (שרשרת assignments)
            └─(נשאלה בשידור)→ used
```

חוקים:
- **רק** `verified` ניתן לאישור בודק. **רק** `checker_approved` ניתן לשיבוץ.
- שיבוץ דורש `difficulty` ו-`round_number` מלאים.
- כל מעבר מצב נכתב ל-`audit_log`.
- עריכת נוסח שאלה/תשובות אחרי `verified` מאפסת ל-`draft` ומבטלת את האישור —
  אחרת תיק התשובה מתייחס לשאלה אחרת. אסורה עריכה כזו במצב `assigned`/`used`.

---

## 3. מדיניות "מקור מוסמך"

מקור קביל חייב להיות מ-tier מותר, ו**שני** המקורות חייבים להיות **בלתי תלויים**:
דומיין רשום שונה, ולא אחד שמצטט את השני.

### מותר (tier)
| tier | מה זה |
|---|---|
| `government` | אתרי מדינה ורשויות: `*.gov.il`, `*.gov`, `*.europa.eu`, אתרי עיריות רשמיים |
| `official_body` | הגוף שהנושא עצמו: `rail.co.il`, `iaa.gov.il`, אגודות ספורט, איגודים מקצועיים |
| `statistics` | למ"ס, OECD, World Bank, Eurostat |
| `academic` | `*.ac.il`, `*.edu`, הוצאות אקדמיות, מאמרים שפיטים, DOI |
| `encyclopedia` | אנציקלופדיות ממוסדות בעריכה מקצועית: Britannica, Oxford Reference, אנציקלופדיה אאוריקה, מט"ח |
| `international_org` | UN, WHO, UNESCO, FIFA, IOC, NASA, ESA |
| `major_news` | כלי תקשורת מרכזיים עם מערכת עיתונאית: ynet, N12, הארץ, כאן, BBC, Reuters, AP, NYT, Guardian |
| `museum_archive` | מוזיאונים, ארכיונים לאומיים, ספריות לאומיות |

### אסור — פסילה אוטומטית
- **ויקיפדיה בכל שפה** (`*.wikipedia.org`), וכל נגזרת: wikidata, wikiquote,
  wikivoyage, dbpedia, וכן מראות/מירורים של ויקיפדיה.
- ויקיז פתוחים בכלל: `fandom.com`, `wikia`, `*.wiki`.
- בלוגים, Medium, Substack, אתרי תוכן אישיים.
- פורומים ו-UGC: Reddit, Quora, Stack Exchange, תפוז, פייסבוק, X, טיקטוק,
  יוטיוב (למעט ערוץ רשמי של גוף מוסמך כמקור **משני**).
- אתרי טריוויה, "עובדות מעניינות", חידונים, מצרפי תוכן, תוכן שנראה
  מיוצר-AI, פארמות SEO.
- אתרי מסחר/מודעות, אתרי מתכונים ואורח חיים לא-מערכתיים.
- עבודות תלמידים, presentations, Scribd, Course Hero.

### בדיקות מכניות בצד השרת (לא סומכים על המודל)
1. נרמול URL → חילוץ דומיין רשום (eTLD+1).
2. בדיקה מול `DENY_DOMAINS` ו-`ALLOW_PATTERNS`. דומיין שנפסל → `checked_ok=false`
   והמקור לא נספר.
3. `HEAD`/`GET` לכל URL: סטטוס 2xx. 404 → המקור לא נספר (מונע לינקים מומצאים).
4. דרישת אי-תלות: ≥2 מקורות עם דומיינים רשומים **שונים**.
5. אם נותרו <2 מקורות קבילים → `verdict = insufficient_sources`, גם אם המודל
   טען שאישר.

---

## 4. מנוע הבדיקה האוטומטית

Edge Function: `verify-question`. קלט: `question_id`. מריץ דרך Lovable AI Gateway
מודל עם חיפוש ווב/גראונדינג, מסנן מקורות בצד השרת, וכותב `verifications` +
`verification_sources` + מעדכן את השאלה.

### 4.1 מה המנוע חייב לבדוק
1. **האם התשובה שהכותב בחר נכונה** — וזו הבדיקה המרכזית.
2. **האם יש יותר מתשובה נכונה אחת** → `needs_rewrite`. (דוגמה אמיתית מהשיט:
   "איפה אין תחנת רכבת? אשקלון / כוכב יאיר / אור יהודה / בית שאן" — גם לכוכב
   יאיר וגם לאור יהודה אין תחנה.)
3. **בהירות הניסוח**: עמימות, חוסר עיגון בזמן ("כיום", "נכון ל-2026"), ניסוח
   סובייקטיבי ("הטוב ביותר"), שלילה כפולה, תשובות שאינן זרות הדדית, מפריעים
   שהם למעשה נכונים חלקית.
4. **תיק תשובה**: 2-4 משפטים על התשובה הנכונה (מה היא, עובדה מעניינת אחת,
   עיגון גיאוגרפי/היסטורי), ושורה אחת לכל מפריע — כמו "ירושלים — בירת ישראל".
5. **הצעת דירוג קושי 1-5** (המלצה; ההחלטה הסופית של הכותב/עורך).

### 4.2 סכמת הפלט (JSON מחייב)

```json
{
  "verdict": "approved | rejected | needs_rewrite | insufficient_sources",
  "confidence": 0.0,
  "writer_choice_correct": true,
  "found_correct_index": 1,
  "multiple_correct": false,
  "multiple_correct_indexes": [],
  "reasoning_he": "הסבר בעברית, 1-3 משפטים, מדוע זו המסקנה",
  "rejection_reason_he": null,
  "clarity_issues_he": [],
  "suggested_question_text_he": null,
  "dossier": {
    "correct_blurb_he": "2-4 משפטים על התשובה הנכונה",
    "distractors": [
      { "index": 1, "one_liner_he": "שורה אחת" }
    ]
  },
  "sources": [
    {
      "url": "https://...",
      "publisher": "שם הגוף",
      "title": "כותרת הדף",
      "tier": "government",
      "quote": "הציטוט שמוכיח את התשובה",
      "supports_index": 3
    }
  ],
  "difficulty_suggestion": 3
}
```

### 4.3 הפרומפט של המנוע (System)

```
אתה בודק עובדות עבור שעשועון טריוויה בטלוויזיה. מטרתך היחידה: לקבוע אם התשובה
שהכותב סימן כנכונה היא אכן נכונה וחד-משמעית, ולגבות זאת בשני מקורות מוסמכים
ובלתי תלויים.

כללי מקורות — מחייבים:
- אסור בהחלט להסתמך על ויקיפדיה בכל שפה, על כל ויקי פתוח (fandom וכו'), על
  בלוגים, פורומים, רשתות חברתיות, אתרי טריוויה או מצרפי תוכן.
- מותר: אתרי ממשלה ורשויות, הגוף הרשמי שהנושא עצמו (למשל רכבת ישראל לשאלה על
  רכבות), למ"ס וגופי סטטיסטיקה, מקורות אקדמיים, אנציקלופדיות ממוסדות בעריכה
  מקצועית (Britannica וכדומה), ארגונים בינלאומיים, כלי תקשורת מרכזיים,
  מוזיאונים וארכיונים לאומיים.
- שני המקורות חייבים להיות מדומיינים שונים ובלתי תלויים זה בזה.
- החזר רק כתובות URL שראית בתוצאות החיפוש וקראת. אל תמציא כתובות. אל תנחש
  כתובת "סבירה". אם אין לך שני מקורות קבילים — החזר verdict =
  "insufficient_sources".

כללי הכרעה:
- אם התשובה שהכותב בחר שגויה → verdict = "rejected", ובשדה rejection_reason_he
  כתוב בדיוק מה נמצא ומה התשובה הנכונה, עם המקור.
- אם התשובה שהכותב בחר נכונה אך גם תשובה אחרת נכונה → multiple_correct = true,
  verdict = "needs_rewrite". זהו מצב פסילה: בשעשועון אסור שיהיו שתי תשובות
  נכונות.
- אם השאלה עמומה, לא מעוגנת בזמן, סובייקטיבית, או שהתשובות אינן זרות הדדית →
  verdict = "needs_rewrite", ובשדה suggested_question_text_he הצע ניסוח מדויק
  שפותר את הבעיה בלי לשנות את כוונת הכותב ובלי לשנות את התשובות.
- אם התשובה נכונה אך לא מדויקת (למשל הניסוח נכון רק בקירוב, או נכון לתאריך
  אחר) → "rejected" עם הסבר מדויק.
- רק אם התשובה נכונה, יחידה, והשאלה ברורה → "approved".
- אם אתה לא בטוח, אל תאשר. confidence נמוך מ-0.75 אינו מצדיק approved.

תיק התשובה:
- correct_blurb_he: 2 עד 4 משפטים בעברית על התשובה הנכונה — מה היא, עובדה אחת
  מעניינת שניתן להקריא בשידור, ועיגון (מקום/תאריך/גוף).
- לכל תשובה לא נכונה: שורה אחת קצרה שמסבירה מה היא כן, בסגנון
  "ירושלים — בירת ישראל".

כל הטקסט לקהל (reasoning_he, rejection_reason_he, clarity_issues_he,
suggested_question_text_he, הבלורבים) בעברית, גם אם המקורות באנגלית.
החזר JSON בלבד לפי הסכמה שניתנה. בלי טקסט מסביב.
```

### 4.4 הפרומפט של המנוע (User)

```
שאלה: {{question.text}}
קטגוריה: {{question.category}}
כותב: {{writer.display_name}}
תשובות:
1. {{option_1}}
2. {{option_2}}
3. {{option_3}}
4. {{option_4}}
התשובה שהכותב סימן כנכונה: מספר {{correct_index}} — "{{correct_option_text}}"
הערות הכותב: {{writer_notes}}
תאריך הבדיקה: {{today}}

חפש בווב, אמת, והחזר JSON לפי הסכמה.
```

### 4.5 אחרי שהמודל חזר — צד שרת
1. ולידציה של ה-JSON מול הסכמה. כשל → `verifying` נשאר, שגיאה ב-`raw_response`.
2. סינון דומיינים + בדיקת 2xx על כל URL (§3).
3. פחות מ-2 מקורות קבילים או `confidence < 0.75` → `insufficient_sources`
   (דורס `approved` של המודל).
4. כתיבת `verifications`, `verification_sources`, `question_options.one_liner`,
   `questions.dossier_summary`, `status`, `suggested_text`, `rejection_reason`.
5. `audit_log`.
6. **אין Rate limit עקיף**: תור עבודה (`verifying` + `claimed_at`) כדי שכפתור
   "בדוק הכל" לא יפציץ את ה-API. בדיקה חוזרת יוצרת `verification` חדש ולא
   דורסת היסטוריה.

### 4.6 Golden set — חובה לפני שחרור
| שאלה | תשובת הכותב | הפלט הנכון |
|---|---|---|
| איפה אין תחנת רכבת? אשקלון/כוכב יאיר/אור יהודה/בית שאן | 3 | `needs_rewrite`, `multiple_correct=true` (גם 2 וגם 3) |
| מהי בירת איטליה? ירושלים/רומא/אתונה | 2 | `approved` + 2 מקורות + תיק תשובה |
| מהי בירת איטליה? ירושלים/רומא/אתונה | 1 | `rejected`, "התשובה הנכונה היא רומא" |
| מי הכי מצחיק? א/ב/ג | 1 | `needs_rewrite` — שאלה סובייקטיבית |
| מי ראש הממשלה? (ללא עיגון בזמן) | — | `needs_rewrite` + הצעת ניסוח עם "נכון ל-…" |
| שאלה על נושא שולי שאין עליו מקורות | — | `insufficient_sources` |

---

## 5. הרשאות (RLS)

פונקציה `has_role(uuid, app_role)` מסוג `security definer` — לעולם לא קוראים
תפקיד מתוך ה-JWT או מטבלת profiles.

| יכולת | מנהל | בודק | כותב שאלות |
|---|---|---|---|
| ייבוא מגוגל שיט / CSV | ✔ | ✘ | ✔ |
| יצירה/עריכת שאלה | הכל | ✘ | רק שלו, ורק במצבי draft / needs_rewrite / rejected |
| בדיקה ראשונה של הכותב (`writer_reviewed`) | ✔ | ✘ | רק שלו |
| הפעלת בדיקה אוטומטית | ✔ | ✔ | רק שלו |
| צפייה בכל השאלות ובתיקי התשובה | ✔ | ✔ | רק שלו |
| שכתוב / בחירת הניסוח שהמערכת הציעה | ✔ | ✘ | רק שלו |
| סימון "לחשיבה על ניסוח מחדש" | ✔ | ✘ | רק שלו |
| שיוך שאלה לכותב אחר | ✔ | ✘ | ✔ |
| אישור ידני "נבדק ע"י בודק" | ✔ | ✔ | ✘ |
| קביעת דירוג קושי | ✔ | ✔ | ✔ (שלו) |
| ניהול מתמודדים | ✔ | ✘ | ✘ |
| שיבוץ שאלה למתמודד | ✔ (עורך) | ✘ | ✘ |
| העברת שיבוץ | ✔ | ✘ | ✘ |
| מחיקת שיבוץ | ✘ | ✘ | ✘ |
| ניהול משתמשים ותפקידים | ✔ | ✘ | ✘ |
| צפייה ב-audit log | ✔ | ✔ (קריאה) | ✘ |

הערה: "כותב יכול לבדוק בדיקה ראשונה" = סימון שעבר על השאלה בעצמו לפני שליחה
לאימות אוטומטי. זה לא תחליף לאישור הבודק.

---

## 6. ייבוא מגוגל שיט

### 6.1 מבנה השיט (מהדוגמה שסופקה)

| עמודה | שדה | הערות |
|---|---|---|
| כותב השאלה | `writers.display_name` | נוצר אוטומטית אם לא קיים |
| קטגוריה | `category` | |
| שאלה | `text` | |
| תשובה 1 | `options[1]` | |
| תשובה 2 | `options[2]` | |
| תשובה 3 | `options[3]` | ריק → שאלת 2 תשובות |
| תשובה 4 | `options[4]` | ריק → שאלת 3 תשובות |
| **תשובה נכונה** | `correct_index` | **אינדקס 1-based, לא טקסט** |
| הערות | `writer_notes` | |

`option_count` = מספר עמודות התשובה הלא-ריקות. עמודות נוספות/נגררות בשיט —
מתעלמים.

### 6.2 שתי דרכי ייבוא
1. **הדבקת/העלאת CSV** — הדרך העיקרית, עובדת תמיד. "קובץ → הורדה → CSV" מגוגל
   שיט. מנתח CSV עם תמיכה בעברית ו-UTF-8 BOM.
2. **קישור לשיט מפורסם** — הדבקת URL של גוגל שיט. המערכת מחלצת את מזהה הגמ"ח
   ומביאה `/export?format=csv&gid=…`. עובד רק אם השיט מפורסם/ציבורי. אם לא —
   שגיאה מנומקת שמפנה לדרך 1, לא כשל אילם.

### 6.3 תצוגה מקדימה לפני ייבוא
טבלת preview עם שורה-שורה: ✔ תקין / ⚠ אזהרה / ✘ שגיאה. חוסם ייבוא על:
- `correct_index` ריק, לא מספר, או גדול ממספר התשובות.
- שאלה או תשובה 1/2 ריקות.
- "חור" בתשובות (תשובה 3 ריקה אבל 4 מלאה).
- כפילות: שאלה זהה (נרמול רווחים) שכבר קיימת → אזהרה + דילוג.
ייבוא הוא טרנזקציה אחת עם `import_batch_id`, וניתן לבטל batch שלם כל עוד אף
שאלה בו לא שובצה.

---

## 7. מסכי המערכת

1. **התחברות** — Supabase Auth, אימייל+סיסמה. משתמש ראשון = admin.
2. **לוח בקרה** — מנייני מצבים (ממתינות לבדיקה, נפסלו, דורשות ניסוח, מאושרות,
   שובצו), פילוח לפי כותב וקטגוריה, כיסוי לפי סיבוב (כמה שאלות מאושרות יש לכל
   סיבוב 1-7 — זה מה שהעורך באמת צריך לדעת).
3. **מחסן השאלות** — טבלה עם סינון לפי מצב / כותב / קטגוריה / סיבוב / קושי,
   חיפוש טקסט, ובחירה מרובה לפעולות אצווה ("שלח לבדיקה", "שייך לכותב").
4. **כרטיס שאלה** — מסך העבודה המרכזי:
   - נוסח, תשובות עם סימון הנכונה, קטגוריה, כותב, קושי, סיבוב.
   - **תיק תשובה**: הבלורב על התשובה הנכונה, שורה לכל מפריע, ורשימת המקורות
     כקישורים עם שם הגוף, ה-tier והציטוט.
   - אם `rejected`: תיבה אדומה עם `rejection_reason`.
   - אם `needs_rewrite`: הצגת הניסוח הקיים מול הניסוח המוצע זה לצד זה, ושני
     כפתורים — "אשר את הניסוח המוצע" / "לחשיבה על ניסוח מחדש". בנוסף עריכה
     חופשית.
   - טיימליין: כל הבדיקות, האישורים, השיבוצים וההעברות.
5. **ייבוא** — §6.
6. **תור בדיקה (בודק)** — רשימת `verified` לאישור ידני, עם כל הראיות מול העיניים
   וכפתור "נבדק ואושר" + הערה.
7. **מתמודדים ושיבוץ (עורך/מנהל)** — ניהול מתמודדים, שיבוץ שאלה
   `checker_approved` למתמודד. פעולת "העברה" דורשת נימוק. אין כפתור ביטול
   שיבוץ — בכלל.
8. **ניהול (מנהל)** — משתמשים ותפקידים, כותבים, קטגוריות, audit log.

---

## 8. UX ועברית
- `dir="rtl"`, `lang="he"` על ה-root. כל הקומפוננטות של shadcn נבדקות ב-RTL.
- כל טקסט המוצג למשתמש בעברית, כולל שגיאות ומצבי ריק. אין מחרוזות אנגלית ב-UI.
- מצבי מצב בצבע: ממתין=אפור, בבדיקה=כחול, אושר=ירוק, דורש ניסוח=כתום,
  נפסל=אדום, שובץ=סגול.
- תוויות קושי: 1 קל, 2 בינוני-קל, 3 בינוני, 4 קשה, 5 קשה מאוד.
- הדפסה/ייצוא: ייצוא CSV וייצוא "דף עורך" להדפסה — שאלה + תשובות + תיק תשובה
  + המתמודד המשובץ.

## 9. מה לא בגרסה הראשונה
ניהול שידור חי, ניקוד, מסך תצוגה לקהל, אפליקציית שלט למתמודד, תרגום שאלות,
העלאת מדיה (תמונה/סרטון בשאלה). הסכמה לא חוסמת הוספה שלהם בהמשך.
