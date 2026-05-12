# Master Plan: Smart Bilingual Reader

**نام کاری محصول:** Smart Bilingual Reader / MirrorRead  
**نوع محصول:** کتاب‌خوان دوزبانه هوشمند برای EPUB  
**زبان مقصد پیش‌فرض:** فارسی  
**Scope مهم:** بدون User Management، بدون Auth، بدون Permissions، بدون Payment، بدون Multi-user Collaboration

---

## فهرست

1. [خلاصه اجرایی](#1-خلاصه-اجرایی)
2. [اصل معماری محصول](#2-اصل-معماری-محصول)
3. [باگ‌های لاجیکالی که باید از اول اصلاح شوند](#3-باگهای-لاجیکالی-که-باید-از-اول-اصلاح-شوند)
4. [تعریف محصول](#4-تعریف-محصول)
5. [Scope و Non-goals](#5-scope-و-non-goals)
6. [مدل مفهومی درست: ReadingUnit به‌جای Page](#6-مدل-مفهومی-درست-readingunit-بهجای-page)
7. [EPUB Pipeline](#7-epub-pipeline)
8. [مدل Block و ساختار محتوا](#8-مدل-block-و-ساختار-محتوا)
9. [Inline Mark Preservation: حفظ Bold/Italic/Link بدون سپردن HTML به AI](#9-inline-mark-preservation-حفظ-bolditaliclink-بدون-سپردن-html-به-ai)
10. [Translation Engine](#10-translation-engine)
11. [Context Pack](#11-context-pack)
12. [Prompting و Structured Output](#12-prompting-و-structured-output)
13. [Validation، Circuit Breaker و Fallback](#13-validation-circuit-breaker-و-fallback)
14. [Caching Strategy](#14-caching-strategy)
15. [Glossary زنده و Retroactive Update](#15-glossary-زنده-و-retroactive-update)
16. [Reader UX](#16-reader-ux)
17. [Sync Scroll، Proportional Scroll و Cross-Highlighting](#17-sync-scroll-proportional-scroll-و-cross-highlighting)
18. [Skeleton Loading و Perceived Performance](#18-skeleton-loading-و-perceived-performance)
19. [Cost Controller](#19-cost-controller)
20. [Queue System و Background Jobs](#20-queue-system-و-background-jobs)
21. [معماری فنی](#21-معماری-فنی)
22. [Data Model پیشنهادی](#22-data-model-پیشنهادی)
23. [API Design](#23-api-design)
24. [ساختار پروژه](#24-ساختار-پروژه)
25. [Settings و Configuration](#25-settings-و-configuration)
26. [Roadmap اجرایی](#26-roadmap-اجرایی)
27. [MVP دقیق](#27-mvp-دقیق)
28. [معیارهای پذیرش](#28-معیارهای-پذیرش)
29. [تست، کیفیت و Observability](#29-تست-کیفیت-و-observability)
30. [ریسک‌ها و تصمیم‌های معماری](#30-ریسکها-و-تصمیمهای-معماری)
31. [نسخه نهایی خلاصه‌شده](#31-نسخه-نهایی-خلاصهشده)

---

# 1. خلاصه اجرایی

Smart Bilingual Reader یک پلتفرم کتاب‌خوان تحت وب است که فایل EPUB را دریافت می‌کند، آن را به ساختار قابل پردازش تبدیل می‌کند، متن اصلی را در کنار ترجمه فارسی نمایش می‌دهد و تجربه خواندن را به‌صورت دوزبانه، ساختارمند و همگام ارائه می‌کند.

هدف محصول این نیست که فقط متن کتاب را به فارسی تبدیل کند. هدف اصلی این است که کاربر بتواند کتاب را با ساختار اصلی بخواند و در کنار آن ترجمه‌ای روان، دقیق، قابل بازتولید و قابل کنترل داشته باشد.

محصول باید این قابلیت‌ها را فراهم کند:

- آپلود EPUB.
- استخراج metadata، cover، TOC، spine و assets.
- پاک‌سازی و نرمال‌سازی XHTML داخل EPUB.
- تقسیم کتاب به ReadingUnitهای پایدار.
- استخراج blockهای معنایی مثل heading، paragraph، image، table، list، footnote و caption.
- نمایش دو پنل همگام: متن اصلی و ترجمه فارسی.
- ترجمه هر ReadingUnit با ContextPack کنترل‌شده.
- حفظ ساختار کتاب توسط اپلیکیشن، نه توسط LLM.
- استفاده از LLM فقط برای ترجمه، پیشنهاد glossary و فهم context.
- cache چندلایه برای کاهش هزینه و افزایش سرعت.
- queue برای ترجمه background و prefetch محدود.
- مدیریت هزینه، budget و انتخاب model/provider.
- glossary زنده با امکان اصلاح اصطلاحات و اعمال تغییرات روی ترجمه‌های قبلی.
- skeleton loading ساختاریافته برای تجربه سریع‌تر.
- sync scroll هوشمند و cross-highlighting در فازهای بعدی.

مهم‌ترین تصمیم معماری:

> **Page محور اصلی سیستم نیست. ReadingUnit محور اصلی سیستم است.**

چون EPUB معمولاً reflowable است و مفهوم صفحه ثابت ندارد. صفحه در reader با توجه به عرض پنجره، سایز فونت، line-height، theme و دستگاه کاربر تغییر می‌کند. بنابراین همه چیز باید حول ReadingUnit و Block ساخته شود، نه Page.

---

# 2. اصل معماری محصول

اصل مرکزی محصول:

> **AI مترجم است، نه parser، نه renderer، نه layout engine، نه state manager.**

اپلیکیشن باید کارهای deterministic را انجام دهد:

- parse کردن EPUB.
- تشخیص ترتیب spine.
- استخراج DOM.
- پاک‌سازی HTML.
- ساخت blockها.
- ساخت ReadingUnit.
- ساخت context.
- کنترل cache.
- mapping بین original و translation.
- sync scroll.
- rendering.
- validation.
- retry.
- queue.
- ذخیره‌سازی.

AI فقط این کارها را انجام می‌دهد:

- ترجمه متن.
- حفظ لحن و معنا.
- حفظ اصطلاحات.
- پیشنهاد اصطلاحات glossary.
- کمک به رفع ابهام زبانی.
- ترجمه table cell، caption، footnote، list item و paragraph در چارچوب schema.

AI نباید این کارها را انجام دهد:

- تولید HTML نهایی.
- ساخت layout.
- تغییر blockId.
- اضافه یا حذف block.
- جابه‌جایی ترتیب متن.
- parse EPUB.
- تصمیم‌گیری درباره assetها.
- پردازش کل کتاب در یک request.
- نگهداری state اصلی سیستم.

---

# 3. باگ‌های لاجیکالی که باید از اول اصلاح شوند

## 3.1 استفاده از Page به‌عنوان واحد ترجمه

### مشکل

EPUB صفحه ثابت ندارد. صفحه در reader وابسته به شرایط نمایش است:

- سایز فونت.
- line-height.
- عرض پنجره.
- device.
- column width.
- theme.
- zoom.
- نمایش original only یا split view.

اگر translation cache، progress، sync و context روی page بنا شوند، بعداً با تغییر تنظیمات reader همه چیز ناپایدار می‌شود.

### راه‌حل

واحد اصلی باید `ReadingUnit` باشد.

```txt
Book
 └── SpineItem
      └── ReadingUnit
           └── Block
```

Page فقط یک مفهوم نمایشی است و در MVP حتی نیازی به ذخیره آن نیست.

---

## 3.2 کش ترجمه بر اساس Page ID

### مشکل

وقتی page پایدار نیست، cache page هم پایدار نیست.

### راه‌حل

cache باید بر اساس این فیلدها ساخته شود:

```txt
bookId
unitId
unitSourceHash
targetLanguage
provider
model
styleHash
glossaryHash
promptVersion
schemaVersion
parserVersion
contextSignature
```

---

## 3.3 ترجمه block بدون context

### مشکل

یک پاراگراف ممکن است در contextهای مختلف ترجمه متفاوتی داشته باشد. اگر block cache بی‌قید استفاده شود، اصطلاحات و ارجاعات به‌هم می‌ریزند.

### راه‌حل

- cache اصلی: unit-level.
- cache کمکی: block-level فقط برای heading، caption، table cell، footnote کوتاه و متن‌های تکراری.
- برای paragraphهای مهم، unit context باید بخشی از cache key باشد.

---

## 3.4 سپردن HTML کامل به مدل

### مشکل

اگر HTML کامل به مدل داده شود و خروجی HTML خواسته شود، مدل ممکن است:

- تگ حذف کند.
- nesting را خراب کند.
- attribute اضافه کند.
- table را خراب کند.
- link را تغییر دهد.
- HTML نامعتبر تولید کند.

### راه‌حل

مدل نباید HTML نهایی تولید کند. در عوض:

- اپلیکیشن DOM را parse می‌کند.
- blockها را استخراج می‌کند.
- inline marks مثل bold/italic/link را به representation امن تبدیل می‌کند.
- مدل فقط text یا segmentهای دارای mark id را ترجمه می‌کند.
- اپلیکیشن خودش HTML/React نهایی را render می‌کند.

---

## 3.5 JSON streaming مستقیم به UI

### مشکل

Streaming JSON معمولاً تا وقتی کامل نشده قابل اعتماد نیست. اگر وسط stream قطع شود، JSON ناقص می‌ماند.

### راه‌حل

- برای UI از SSE یا WebSocket برای status/progress استفاده شود.
- ترجمه می‌تواند internal streaming داشته باشد، اما فقط بعد از validation نهایی ذخیره و render شود.
- اگر block-level streaming لازم شد، هر block باید event مستقل معتبر داشته باشد، نه یک JSON بزرگ نیمه‌کاره.

---

## 3.6 Glossary بدون versioning

### مشکل

اگر کاربر ترجمه اصطلاحی را عوض کند، ترجمه‌های قبلی ممکن است outdated شوند.

### راه‌حل

- هر glossary وضعیت و نسخه داشته باشد.
- فقط termهای `approved` وارد prompt شوند.
- تغییر glossary باعث `stale` شدن translationهای وابسته شود.
- برای retroactive update، سیستم فقط unitهای متاثر را دوباره ترجمه یا patch کند.

---

# 4. تعریف محصول

## 4.1 یک جمله‌ای

Smart Bilingual Reader یک EPUB reader دوزبانه است که کتاب را ساختارمند parse می‌کند، متن را به واحدهای پایدار تقسیم می‌کند، هر واحد را با context کنترل‌شده ترجمه می‌کند و متن اصلی و ترجمه فارسی را به‌صورت همگام و قابل مقایسه نمایش می‌دهد.

## 4.2 ارزش اصلی

### ارزش 1: ترجمه با context، بدون ارسال کل کتاب

برای ترجمه هر ReadingUnit، فقط اطلاعات لازم ارسال می‌شود:

- متن target unit.
- excerpt از unit قبل.
- excerpt از unit بعد.
- عنوان فصل.
- مسیر headingها.
- metadata کتاب.
- glossary فعال.
- summary کوتاه فصل، اگر موجود باشد.
- named entities مهم.
- translation memory مرتبط.

مدل فقط target unit را ترجمه می‌کند.

---

### ارزش 2: حفظ ساختار توسط app، نه AI

اپلیکیشن EPUB را parse می‌کند و فقط blockهای قابل ترجمه را به مدل می‌دهد. مدل نباید HTML نهایی بسازد.

نمونه ورودی امن به AI:

```json
{
  "unitId": "unit_0042",
  "blocks": [
    {
      "blockId": "b_1001",
      "type": "heading",
      "text": "The Architecture of Memory"
    },
    {
      "blockId": "b_1002",
      "type": "paragraph",
      "segments": [
        { "segmentId": "s1", "text": "Memory is not a " },
        { "segmentId": "s2", "text": "single system", "marks": ["m1"] },
        { "segmentId": "s3", "text": "." }
      ],
      "marks": [
        { "markId": "m1", "type": "italic" }
      ]
    },
    {
      "blockId": "b_1003",
      "type": "image",
      "translatable": false
    }
  ]
}
```

نمونه خروجی:

```json
{
  "unitId": "unit_0042",
  "translations": [
    {
      "blockId": "b_1001",
      "type": "text",
      "translatedText": "معماری حافظه"
    },
    {
      "blockId": "b_1002",
      "type": "segmented_text",
      "segments": [
        { "segmentId": "s1", "translatedText": "حافظه یک " },
        { "segmentId": "s2", "translatedText": "نظام واحد", "marks": ["m1"] },
        { "segmentId": "s3", "translatedText": " نیست." }
      ]
    }
  ]
}
```

سپس app خودش متن فارسی را با markهای درست render می‌کند.

---

### ارزش 3: تجربه خواندن دوزبانه واقعی

محصول باید برای خواندن واقعی ساخته شود، نه فقط ترجمه batch:

- split view.
- translation only.
- original only.
- interleaved view.
- sync scroll.
- cross-highlight.
- glossary tooltip.
- footnote popover.
- skeleton loading.
- progress.
- note/highlight/bookmark.
- regenerate translation.
- improve selected block.

---

### ارزش 4: هزینه قابل کنترل

چون استفاده از APIهای AI هزینه دارد:

- cache اجباری است.
- prefetch باید محدود باشد.
- ترجمه کل کتاب در MVP نباید default باشد.
- user باید بتواند model/provider/style را کنترل کند.
- داشبورد هزینه لازم است.
- daily budget/token limit لازم است.

---

# 5. Scope و Non-goals

## 5.1 داخل Scope

- Upload EPUB.
- Validate EPUB.
- Extract metadata.
- Extract cover.
- Extract TOC.
- Extract spine.
- Extract assets.
- Sanitize XHTML.
- Normalize DOM/CSS.
- Extract semantic blocks.
- Build ReadingUnits.
- Render original book.
- Render translated book.
- Translate current ReadingUnit.
- Use context from adjacent units.
- Use glossary.
- Cache translations.
- Queue background translations.
- Prefetch next unit.
- Skeleton loading.
- Sync scroll.
- Basic proportional scroll.
- Glossary suggestions.
- Cost dashboard.
- Workspace-level settings.
- Docker Compose.
- Postgres.
- Redis.
- Local storage or MinIO.

## 5.2 خارج از Scope

- User management.
- Login/logout.
- Auth.
- Permissions.
- Roles.
- Teams.
- Payment.
- Subscription.
- Mobile app.
- PDF support.
- OCR.
- DRM EPUB.
- Collaboration.
- Real-time multi-user annotations.
- Export EPUB/PDF in MVP.
- Full automatic book translation in MVP.
- Fine-tuning.
- Vector search در MVP.
- Marketplace.
- Public sharing.

---

# 6. مدل مفهومی درست: ReadingUnit به‌جای Page

## 6.1 چرا Page اشتباه است؟

در EPUB، مخصوصاً EPUBهای reflowable، صفحه یک چیز ثابت نیست. صفحه بعد از render در browser ساخته می‌شود. بنابراین این موارد نباید بر اساس Page ذخیره شوند:

- translation.
- cache.
- progress.
- annotation.
- glossary references.
- sync mapping.
- context.

## 6.2 ReadingUnit چیست؟

ReadingUnit یک قطعه پایدار و منطقی از کتاب است که:

- درخت DOM را بی‌دلیل نمی‌شکند.
- وسط paragraph یا sentence را قطع نمی‌کند.
- حدوداً ۸۰۰ تا ۱۲۰۰ کلمه دارد.
- headingهای مرتبط را حفظ می‌کند.
- table یا figure را در جای درست نگه می‌دارد.
- ID پایدار دارد.
- مبنای translation و cache است.

## 6.3 ساختار سطح بالا

```txt
Book
 ├── Metadata
 ├── Assets
 ├── TOC
 └── SpineItem[]
      └── ReadingUnit[]
           └── Block[]
                └── InlineSegment[]
```

## 6.4 Page در UI

در UI ممکن است بعداً Page-like pagination داشته باشیم، اما آن صرفاً layer نمایشی است:

```txt
VisualPage = rendering artifact
ReadingUnit = data and translation unit
```

---

# 7. EPUB Pipeline

EPUB Pipeline مهم‌ترین بخش backend است. کیفیت ترجمه و UX به کیفیت این pipeline وابسته است.

## 7.1 Pipeline کلی

```txt
Upload EPUB
  ↓
Validate EPUB
  ↓
Store original file
  ↓
Extract EPUB archive
  ↓
Find package document
  ↓
Read metadata
  ↓
Read manifest
  ↓
Read spine order
  ↓
Read navigation / TOC
  ↓
Extract assets
  ↓
Sanitize XHTML
  ↓
Normalize DOM and CSS
  ↓
Extract semantic blocks
  ↓
Resolve images, captions, tables, footnotes
  ↓
Build ReadingUnits
  ↓
Persist database records
  ↓
Book status = ready
```

---

## 7.2 Validation

Validation حداقلی:

- فایل zip معتبر باشد.
- mimetype درست باشد.
- container/package document پیدا شود.
- manifest قابل خواندن باشد.
- spine خالی نباشد.
- content documents قابل parse باشند.
- فایل DRM-protected نباشد یا حداقل قابل پردازش نباشد.
- asset paths resolve شوند.

وضعیت‌های پیشنهادی:

```ts
type BookProcessingStatus =
  | "uploaded"
  | "validating"
  | "extracting"
  | "parsing_metadata"
  | "parsing_spine"
  | "sanitizing"
  | "normalizing"
  | "extracting_blocks"
  | "building_units"
  | "ready"
  | "failed";
```

---

## 7.3 Sanitization

هدف sanitization این نیست که ظاهر کتاب نابود شود. هدف این است که HTML قابل کنترل، امن و قابل render شود.

### حذف شود

- `<script>`
- event handlers مثل `onclick`
- remote scripts
- inline JS
- styleهای خطرناک یا مزاحم
- iframe
- object/embed
- CSS fixed positioning نامناسب
- backgroundهای hardcoded که dark mode را خراب می‌کنند
- font-sizeهای fixed خیلی عجیب
- چندین `<br>` پشت سر هم که نقش paragraph دارند

### حفظ شود

- structure.
- headings.
- paragraphs.
- lists.
- tables.
- images.
- captions.
- links.
- emphasis.
- strong.
- sub/sup.
- footnote references.
- semantic tags.

### مثال تبدیل

ورودی:

```html
<p style="font-size:13px;color:#000;background:#fff">
  Hello<br><br>World
</p>
```

خروجی قابل کنترل:

```html
<p class="epub-p">Hello</p>
<p class="epub-p">World</p>
```

---

## 7.4 CSS Normalization

هدف: reader بتواند typography را کنترل کند.

### کارهای لازم

- تبدیل font-sizeهای fixed به scale قابل کنترل.
- حذف رنگ‌های hardcoded مزاحم.
- تبدیل `margin-left/right` به logical properties در صورت امکان.
- آماده‌سازی RTL.
- جلوگیری از overflow table/image.
- نرمال کردن line-height.
- حفظ classهای لازم برای layout.

### نمونه logical direction

به‌جای:

```css
margin-left: 2rem;
```

استفاده از:

```css
margin-inline-start: 2rem;
```

---

## 7.5 DOM-Based Chunking

کتاب نباید صرفاً با word count بریده شود.

### قواعد chunking

- نقطه برش فقط در انتهای nodeهای سطح بالا.
- paragraph وسط کار شکسته نشود.
- heading با محتوای بعدش جدا نشود.
- figure با caption خودش بماند.
- footnote نزدیک reference خودش یا حداقل linked باقی بماند.
- table بزرگ unit مستقل شود.
- unitها حدوداً ۸۰۰ تا ۱۲۰۰ کلمه باشند.
- اگر یک paragraph بسیار بلند بود، split با sentence boundary انجام شود و metadata offset ذخیره شود.

### الگوریتم ساده

```txt
currentUnit = []
currentWordCount = 0

for block in spineBlocks:
  if block is heading and currentUnit is not empty:
      close currentUnit if it has enough content

  if currentWordCount + block.wordCount > maxWords:
      if currentUnit.wordCount >= minWords:
          close currentUnit
      else:
          add block anyway

  add block to currentUnit

close remaining unit
```

---

# 8. مدل Block و ساختار محتوا

## 8.1 BlockBase

```ts
type BlockBase = {
  id: string;
  bookId: string;
  spineItemId: string;
  unitId: string;
  order: number;
  sourceHash: string;
  originalHref: string;
  cfiStart?: string;
  cfiEnd?: string;
  parentHeadingIds: string[];
  metadata?: Record<string, unknown>;
};
```

## 8.2 Block Types

```ts
type HeadingBlock = BlockBase & {
  type: "heading";
  level: 1 | 2 | 3 | 4 | 5 | 6;
  text: string;
  inlineSegments?: InlineSegment[];
};

type ParagraphBlock = BlockBase & {
  type: "paragraph";
  text: string;
  inlineSegments?: InlineSegment[];
};

type QuoteBlock = BlockBase & {
  type: "quote";
  text: string;
  citation?: string;
  inlineSegments?: InlineSegment[];
};

type ListBlock = BlockBase & {
  type: "list";
  ordered: boolean;
  items: {
    id: string;
    text: string;
    inlineSegments?: InlineSegment[];
    children?: ListBlock["items"];
  }[];
};

type TableBlock = BlockBase & {
  type: "table";
  caption?: string;
  rows: {
    cells: {
      id: string;
      text: string;
      inlineSegments?: InlineSegment[];
      role?: "header" | "data";
      rowSpan?: number;
      colSpan?: number;
    }[];
  }[];
};

type ImageBlock = BlockBase & {
  type: "image";
  assetId: string;
  src: string;
  alt?: string;
  title?: string;
  captionBlockId?: string;
};

type CaptionBlock = BlockBase & {
  type: "caption";
  text: string;
  targetBlockId: string;
  inlineSegments?: InlineSegment[];
};

type FootnoteReferenceBlock = BlockBase & {
  type: "footnote_reference";
  refId: string;
  label: string;
};

type FootnoteBodyBlock = BlockBase & {
  type: "footnote_body";
  refId: string;
  text: string;
  inlineSegments?: InlineSegment[];
};

type CodeBlock = BlockBase & {
  type: "code";
  language?: string;
  code: string;
  translatePolicy: "do_not_translate" | "translate_comments_only";
};

type VerseBlock = BlockBase & {
  type: "verse";
  lines: {
    id: string;
    text: string;
    inlineSegments?: InlineSegment[];
  }[];
};

type DividerBlock = BlockBase & {
  type: "divider";
};
```

---

# 9. Inline Mark Preservation: حفظ Bold/Italic/Link بدون سپردن HTML به AI

ایده دوم می‌گوید target unit شامل HTML تمیز شده و inline tagهایی مثل `<i>` و `<a>` باشد و مدل همان تگ‌ها را حفظ کند. این ایده از نظر محصولی جذاب است، اما از نظر معماری اگر خام اجرا شود ریسک دارد.

راه‌حل بهتر:

> **HTML خام به مدل نده. Inline tagها را به Mark/Segment تبدیل کن.**

## 9.1 مدل InlineSegment

```ts
type InlineMark =
  | {
      id: string;
      type: "bold" | "italic" | "underline" | "sup" | "sub" | "code";
    }
  | {
      id: string;
      type: "link";
      href: string;
      title?: string;
    };

type InlineSegment = {
  id: string;
  text: string;
  markIds: string[];
};
```

مثال:

```html
Memory is not a <i>single system</i>.
```

تبدیل به:

```json
{
  "text": "Memory is not a single system.",
  "marks": [
    {
      "id": "m1",
      "type": "italic"
    }
  ],
  "segments": [
    {
      "id": "s1",
      "text": "Memory is not a ",
      "markIds": []
    },
    {
      "id": "s2",
      "text": "single system",
      "markIds": ["m1"]
    },
    {
      "id": "s3",
      "text": ".",
      "markIds": []
    }
  ]
}
```

خروجی مدل:

```json
{
  "blockId": "b1",
  "type": "segmented_text",
  "segments": [
    {
      "sourceSegmentId": "s1",
      "translatedText": "حافظه یک "
    },
    {
      "sourceSegmentId": "s2",
      "translatedText": "نظام واحد",
      "markIds": ["m1"]
    },
    {
      "sourceSegmentId": "s3",
      "translatedText": " نیست."
    }
  ]
}
```

## 9.2 مزیت

- مدل HTML خراب نمی‌کند.
- لینک‌ها توسط app حفظ می‌شوند.
- app کنترل کامل روی render دارد.
- validation ساده‌تر می‌شود.
- XSS و attribute injection کاهش پیدا می‌کند.
- امکان cross-highlighting در آینده بهتر می‌شود.

## 9.3 نسخه MVP

برای MVP می‌توان inline marks را ساده‌تر کرد:

- bold/italic/link حفظ شود.
- nestingهای پیچیده flatten شوند.
- اگر translation segmented fail شد، fallback به plain translated text.

---

# 10. Translation Engine

## 10.1 فلو ترجمه

```txt
User opens ReadingUnit
  ↓
Check unit translation cache
  ↓
If cache hit:
      render translation
  ↓
If cache miss:
      build ContextPack
      create high-priority job
      send target unit to provider
      validate response
      save translations
      render translation
      enqueue next unit prefetch
```

## 10.2 translatable vs non-translatable

قابل ترجمه:

- heading text.
- paragraph text.
- quote text.
- list item text.
- table cell text.
- caption text.
- footnote text.
- image alt/title در صورت فعال بودن.
- verse lines.

غیرقابل ترجمه:

- image binary.
- CSS.
- font.
- code block به‌صورت پیش‌فرض.
- IDs.
- href.
- asset paths.
- DOM structure.
- block order.

## 10.3 Provider abstraction

```ts
type TranslationProvider = {
  name: string;
  supportsStructuredOutput: boolean;
  supportsStreaming: boolean;
  translateUnit(input: TranslationProviderInput): Promise<TranslationProviderOutput>;
};
```

Providerهای قابل پشتیبانی:

- OpenAI-compatible.
- Anthropic.
- Google.
- OpenRouter.
- Local model.
- Custom base URL.

برای MVP یک provider کافی است، اما abstraction باید از ابتدا درست باشد.

---

# 11. Context Pack

## 11.1 تعریف

ContextPack بسته‌ای سبک و کنترل‌شده است که به مدل کمک می‌کند ترجمه target unit را درست انجام دهد، بدون اینکه کل کتاب ارسال شود.

```ts
type ContextPack = {
  book: {
    title: string;
    author?: string;
    sourceLanguage?: string;
    targetLanguage: string;
  };

  location: {
    spineItemTitle?: string;
    chapterTitle?: string;
    parentHeadings: string[];
    unitOrder: number;
  };

  previousUnit?: {
    unitId: string;
    plainTextExcerpt: string;
    translatedExcerpt?: string;
  };

  nextUnit?: {
    unitId: string;
    plainTextExcerpt: string;
  };

  summaries?: {
    bookSummary?: string;
    chapterSummary?: string;
    previousUnitsSummary?: string;
  };

  glossary: {
    source: string;
    target: string;
    description?: string;
    status: "approved" | "suggested" | "rejected";
  }[];

  namedEntities: {
    source: string;
    preferredRendering?: string;
    type?: "person" | "place" | "organization" | "concept" | "other";
  }[];

  translationMemory?: {
    sourceHash: string;
    sourceText: string;
    translatedText: string;
  }[];

  constraints: {
    maxOutputTokens?: number;
    preserveNames: boolean;
    preserveParagraphBoundaries: boolean;
    translateFootnotes: boolean;
    translateCaptions: boolean;
    translateTables: boolean;
    preserveInlineMarks: boolean;
  };
};
```

## 11.2 Stripped Context

previous/next unit نباید با HTML کامل ارسال شوند. برای context کافی است:

- plain text.
- excerpt محدود.
- بدون image.
- بدون table کامل.
- بدون CSS.
- بدون attributes.

```txt
Target Unit = structured blocks with segments
Previous Unit = stripped plain text
Next Unit = stripped plain text
```

## 11.3 Context Budget

ترتیب اولویت context:

1. target unit کامل.
2. approved glossary مرتبط.
3. chapter title و parent headings.
4. previous unit excerpt.
5. next unit excerpt.
6. chapter summary.
7. named entities.
8. translation memory.

اگر token budget کم بود، اولویت‌های پایین حذف می‌شوند.

---

# 12. Prompting و Structured Output

## 12.1 System Prompt

```txt
You are a professional English-to-Persian book translator.

Translate only the target reading unit.
Use previous and next units only for context.
Do not summarize.
Do not add new content.
Do not remove content.
Do not reorder blocks.
Do not translate IDs.
Do not output HTML.
Preserve meaning, tone, terminology, and paragraph boundaries.
Preserve list item mapping.
Preserve table cell mapping.
Preserve inline mark IDs when provided.
Do not invent mark IDs.
Return only JSON matching the provided schema.
```

## 12.2 Style Modes

```ts
type TranslationStyleMode =
  | "faithful"
  | "natural"
  | "literary"
  | "educational"
  | "technical";
```

### faithful

- وفادار به متن.
- کمترین دخل‌وتصرف.
- مناسب کتاب‌های فلسفی، حقوقی، تاریخی.

### natural

- فارسی روان.
- حفظ معنا با جمله‌بندی طبیعی.
- default پیشنهادی.

### literary

- مناسب رمان، داستان، نثر ادبی.
- روان، خوش‌خوان، با حفظ لحن.

### educational

- کمی شفاف‌تر.
- اصطلاحات دشوار می‌توانند note کوتاه داشته باشند.
- مناسب زبان‌آموزی.

### technical

- دقت اصطلاحات اولویت دارد.
- glossary سخت‌گیرانه‌تر اعمال می‌شود.
- مناسب علوم، مهندسی، پزشکی، اقتصاد.

## 12.3 Translation Request

```ts
type TranslationRequest = {
  task: "translate_reading_unit";
  unitId: string;
  sourceLanguage?: string;
  targetLanguage: "fa";
  style: {
    mode: TranslationStyleMode;
    preserveNames: boolean;
    preserveTerminology: boolean;
    persianFluency: "standard" | "high";
    explanationLevel: "none" | "brief_notes";
  };
  context: ContextPack;
  targetUnit: {
    unitId: string;
    blocks: TranslatableBlockPayload[];
  };
};
```

## 12.4 Translation Response

```ts
type TranslationResponse = {
  unitId: string;

  translations: (
    | {
        blockId: string;
        type: "text";
        translatedText: string;
        notes?: string[];
      }
    | {
        blockId: string;
        type: "segmented_text";
        segments: {
          sourceSegmentId: string;
          translatedText: string;
          markIds?: string[];
        }[];
        notes?: string[];
      }
    | {
        blockId: string;
        type: "list";
        translatedItems: {
          itemId: string;
          translatedText: string;
        }[];
        notes?: string[];
      }
    | {
        blockId: string;
        type: "table";
        translatedCaption?: string;
        translatedCells: {
          cellId: string;
          translatedText: string;
        }[];
        notes?: string[];
      }
    | {
        blockId: string;
        type: "verse";
        translatedLines: {
          lineId: string;
          translatedText: string;
        }[];
        notes?: string[];
      }
  )[];

  detectedTerms?: {
    source: string;
    suggestedPersian: string;
    reason?: string;
    confidence: "low" | "medium" | "high";
  }[];

  warnings?: {
    blockId?: string;
    code:
      | "ambiguous_source"
      | "missing_context"
      | "untranslatable_fragment"
      | "formatting_risk";
    message: string;
  }[];
};
```

---

# 13. Validation، Circuit Breaker و Fallback

## 13.1 Validation

بعد از response مدل:

```txt
1. JSON parse شود.
2. schema validation پاس شود.
3. unitId برابر request باشد.
4. همه blockIdها معتبر باشند.
5. block اضافه وجود نداشته باشد.
6. block ضروری حذف نشده باشد.
7. itemIdها در list حفظ شده باشند.
8. cellIdها در table حفظ شده باشند.
9. sourceSegmentIdها معتبر باشند.
10. markId جدید ساخته نشده باشد.
11. خروجی HTML خام نداشته باشد.
12. ترجمه خالی نباشد، مگر source خالی بوده.
13. detectedTerms مستقیم وارد glossary approved نشوند.
```

## 13.2 Circuit Breaker

```txt
Attempt 1:
  Normal translation

If invalid JSON or schema error:
  Attempt 2:
    Repair prompt + temperature lower

If still invalid:
  Attempt 3:
    Split unit into smaller block groups

If still failed:
  Fallback:
    paragraph-by-paragraph translation

If still failed:
  mark unit translation as failed
```

## 13.3 Error States

```ts
type TranslationFailureCode =
  | "provider_timeout"
  | "rate_limited"
  | "invalid_json"
  | "schema_validation_failed"
  | "missing_blocks"
  | "unsafe_output"
  | "token_limit_exceeded"
  | "unknown_provider_error";
```

## 13.4 UI behavior on failure

- پنل ترجمه خالی نماند.
- skeleton یا error card نشان داده شود.
- retry button نمایش داده شود.
- اگر فقط چند block failed شدند، همان blockها failed باشند، نه کل unit.
- خطای provider قابل مشاهده ولی user-friendly باشد.

---

# 14. Caching Strategy

## 14.1 لایه‌های cache

### Unit Translation Cache

cache اصلی:

```ts
type UnitTranslationCacheKeyInput = {
  bookId: string;
  unitId: string;
  unitSourceHash: string;
  targetLanguage: string;
  provider: string;
  model: string;
  styleHash: string;
  glossaryHash: string;
  promptVersion: string;
  schemaVersion: string;
  parserVersion: string;
  contextSignature: string;
};
```

### Block Cache

cache کمکی برای:

- heading.
- caption.
- footnote کوتاه.
- list item.
- table cell.
- short repeated paragraph.

### Provider Prompt Cache

فقط optimization است. نباید منطق اصلی سیستم به آن وابسته باشد.

## 14.2 Cache invalidation

cache باید invalidate یا stale شود اگر:

- source text تغییر کند.
- parser version تغییر کند.
- segmentation تغییر کند.
- prompt version تغییر کند.
- schema version تغییر کند.
- glossary approved تغییر کند.
- style تغییر کند.
- provider/model تغییر کند.
- user regenerate کند.
- book reprocess شود.

## 14.3 Stale Translation

وقتی glossary تغییر می‌کند، لازم نیست همه چیز فوری retranslate شود. می‌توان status را `stale` کرد:

```txt
Translated but possibly outdated due to glossary change.
```

UI می‌تواند badge نشان دهد:

```txt
Stale — glossary changed
```

---

# 15. Glossary زنده و Retroactive Update

## 15.1 Glossary مدل

```ts
type GlossaryTerm = {
  id: string;
  bookId: string;
  source: string;
  target: string;
  description?: string;
  status: "suggested" | "approved" | "rejected";
  confidence: "low" | "medium" | "high";
  createdBy: "ai" | "manual";
  firstSeenBlockId?: string;
  createdAt: Date;
  updatedAt: Date;
};
```

## 15.2 جریان پیشنهاد اصطلاح

```txt
Translation response includes detectedTerms
  ↓
App saves terms as suggested
  ↓
User sees glossary suggestions
  ↓
User approves / edits / rejects
  ↓
Approved terms enter future ContextPacks
```

## 15.3 Living Glossary

Glossary باید در طول خواندن رشد کند:

- اصطلاحات تخصصی.
- نام شخصیت‌ها.
- مکان‌ها.
- سازمان‌ها.
- ترجمه‌های ترجیحی.
- عبارت‌های تکرارشونده.

## 15.4 Retroactive Update

مثال:

کاربر در فصل ۳ تغییر می‌دهد:

```txt
Working Memory:
حافظه کاری → حافظه فعال
```

سیستم باید بگوید:

```txt
این اصطلاح در ۵ واحد قبلی استفاده شده است.
می‌خواهید ترجمه‌های قبلی به‌روزرسانی شوند؟
```

### دو روش اعمال

#### روش 1: Safe Replace

برای موارد ساده:

```txt
حافظه کاری → حافظه فعال
```

اگر عبارت دقیق و بدون ریسک باشد، replace انجام می‌شود.

#### روش 2: Targeted Retranslation

برای موارد حساس:

- اصطلاح در جمله‌های پیچیده آمده.
- چند ترجمه ممکن دارد.
- ساختار جمله تغییر می‌کند.
- style ادبی است.

در این حالت unitهای متاثر دوباره ترجمه می‌شوند.

## 15.5 وابستگی translation به glossary

هر Translation باید `glossaryHash` داشته باشد. وقتی glossary تغییر کند:

- unitهای وابسته `stale` می‌شوند.
- future translations از glossary جدید استفاده می‌کنند.

---

# 16. Reader UX

## 16.1 Modes

```txt
Original only
Translation only
Split view
Interleaved view
Focus mode
```

## 16.2 Split View

پیشنهاد default:

- Original: سمت چپ.
- Persian translation: سمت راست.

اما باید قابل تنظیم باشد:

- translation right.
- translation left.
- column width.
- gap.
- font size.
- line height.
- theme.

## 16.3 Interleaved View

برای موبایل یا مطالعه آموزشی:

```txt
Original paragraph
Persian translation
Original paragraph
Persian translation
...
```

## 16.4 Reader Toolbar

- TOC.
- Search.
- Theme.
- Font settings.
- Translate current unit.
- Regenerate.
- Toggle sync.
- Toggle focus mode.
- Translation status.
- Cost estimate.
- Glossary.
- Notes/highlights.

## 16.5 Block Actions

وقتی کاربر روی block کلیک می‌کند:

- Copy original.
- Copy translation.
- Regenerate block.
- Improve translation.
- Add to glossary.
- Explain this.
- Highlight.
- Add note.

## 16.6 Empty states

- کتابی وجود ندارد.
- کتاب هنوز processing است.
- کتاب failed شده.
- translation provider تنظیم نشده.
- API key وارد نشده.
- unit ترجمه نشده.
- ترجمه در حال انجام است.
- ترجمه fail شده.
- translation stale است.

---

# 17. Sync Scroll، Proportional Scroll و Cross-Highlighting

## 17.1 Sync Scroll ساده

هر block در original و translation ID مشترک دارد:

```html
<p data-block-id="b_1002">Thought is rarely linear...</p>
<p data-block-id="b_1002">اندیشه به‌ندرت خطی است...</p>
```

با `IntersectionObserver` block فعال تشخیص داده می‌شود و پنل مقابل به همان block scroll می‌شود.

## 17.2 چرا scrollTop کافی نیست؟

طول متن فارسی و انگلیسی متفاوت است. بنابراین raw scrollTop باعث mismatch می‌شود.

## 17.3 Proportional Sync Scroll

ایده:

اگر کاربر ۳۵٪ داخل block انگلیسی پیش رفته، translation pane هم به ۳۵٪ داخل block فارسی برود.

فرمول مفهومی:

```txt
sourceBlockProgress =
  (viewportTop - sourceBlockTop) / sourceBlockHeight

targetScrollTop =
  targetBlockTop + sourceBlockProgress * targetBlockHeight
```

## 17.4 Anti-Jerkiness

برای جلوگیری از پرش:

- scroll animation کوتاه.
- debounce.
- تشخیص اینکه scroll از user آمده یا sync.
- lock کوتاه هنگام sync.
- فقط وقتی active block عوض شد sync شود.
- proportional update برای blockهای بلند.

## 17.5 Cross-Highlighting

فاز پیشرفته:

- hover یا tap روی جمله انگلیسی.
- جمله متناظر فارسی highlight شود.
- بقیه متن کمی dim شود.

### نیاز فنی

برای این قابلیت باید sentence-level mapping ساخته شود.

```ts
type SentenceAlignment = {
  sourceSentenceId: string;
  targetSentenceId: string;
  sourceBlockId: string;
  targetBlockId: string;
  confidence: "low" | "medium" | "high";
};
```

### روش امن

App source text را sentence-segment می‌کند و به مدل sentenceId می‌دهد. مدل باید ترجمه را با همان sentenceId یا target sentence mapping برگرداند.

نباید صرفاً از مدل خواست خودش spanهای مخفی دلخواه بسازد؛ چون ممکن است IDها را خراب کند.

---

# 18. Skeleton Loading و Perceived Performance

وقتی translation هنوز آماده نیست، UI نباید خالی باشد.

چون app ساختار blockهای original را می‌داند، می‌تواند skeleton ترجمه را تقریباً مشابه ساختار اصلی render کند:

- heading skeleton برای heading.
- paragraph skeleton با تعداد line تقریبی.
- table skeleton.
- caption skeleton.
- footnote skeleton.

## 18.1 Skeleton برای blockها

```ts
type SkeletonShape =
  | { type: "heading"; lines: 1; width: "short" | "medium" | "long" }
  | { type: "paragraph"; lines: number }
  | { type: "table"; rows: number; cols: number }
  | { type: "caption"; lines: number };
```

## 18.2 مزیت

- perceived speed بالا می‌رود.
- کاربر می‌فهمد ترجمه در همان ساختار خواهد آمد.
- UI از نظر layout کمتر shift می‌کند.
- سمت راست reader blank نمی‌ماند.

---

# 19. Cost Controller

## 19.1 چرا لازم است؟

ترجمه کتاب با LLM می‌تواند گران شود. محصول باید از ابتدا cost-aware باشد.

## 19.2 قابلیت‌ها

- تخمین token کل کتاب.
- تخمین هزینه ترجمه کل کتاب.
- تخمین هزینه current chapter.
- نمایش هزینه مصرف‌شده.
- نمایش input/output tokens.
- budget limit.
- daily token limit.
- max concurrent jobs.
- انتخاب model.
- انتخاب style.
- خاموش/روشن کردن prefetch.
- فقط ترجمه current unit.
- عدم ترجمه tables/footnotes/captions در صورت نیاز.

## 19.3 Model Presets

```ts
type ModelPreset = {
  id: string;
  label: string;
  provider: string;
  model: string;
  useCase: "cheap" | "balanced" | "quality" | "technical" | "local";
};
```

مثال use case:

```txt
Cheap:
  رمان ساده، خواندن سریع

Balanced:
  کتاب عمومی، non-fiction

Quality:
  ادبیات، فلسفه، متن حساس

Technical:
  کتاب تخصصی با glossary سخت‌گیرانه

Local:
  حریم خصوصی و کنترل هزینه
```

## 19.4 Budget Behavior

اگر budget limit نزدیک شد:

```txt
80%:
  show warning

100%:
  stop background prefetch

110%:
  block new bulk jobs but allow current unit manually
```

---

# 20. Queue System و Background Jobs

## 20.1 چرا queue لازم است؟

ترجمه نباید کل UI را block کند. بعضی کارها باید background انجام شوند:

- parsing.
- translation.
- retry.
- glossary extraction.
- chapter summary.
- stale retranslation.
- prefetch.

## 20.2 Job Types

```ts
type JobType =
  | "parse_book"
  | "translate_unit"
  | "prefetch_unit"
  | "extract_glossary"
  | "summarize_chapter"
  | "retroactive_glossary_update"
  | "cleanup_assets";
```

## 20.3 Priority

```ts
enum TranslationPriority {
  CURRENT_UNIT = 100,
  NEXT_UNIT = 80,
  PREVIOUS_UNIT = 60,
  CURRENT_CHAPTER_PREFETCH = 40,
  NEXT_CHAPTER_IDLE = 20,
  BULK = 10,
  MAINTENANCE = 5
}
```

## 20.4 Current Unit

- highest priority.
- UI shows translating state.
- SSE sends progress events.
- result stored only after validation.

## 20.5 Next Unit

- background prefetch.
- only if enabled.
- only if budget allows.
- only if queue not overloaded.

## 20.6 Next Chapter

- low priority.
- only in idle mode.
- outside MVP or disabled by default.

## 20.7 SSE Events

```ts
type TranslationEvent =
  | {
      type: "queued";
      jobId: string;
      unitId: string;
    }
  | {
      type: "started";
      jobId: string;
      unitId: string;
    }
  | {
      type: "progress";
      jobId: string;
      unitId: string;
      message: string;
    }
  | {
      type: "completed";
      jobId: string;
      unitId: string;
    }
  | {
      type: "failed";
      jobId: string;
      unitId: string;
      errorCode: string;
      message: string;
    };
```

## 20.8 Duplicate Job Prevention

برای یک ترکیب مشخص نباید چند job همزمان ساخته شود:

```txt
bookId
unitId
targetLanguage
provider
model
styleHash
glossaryHash
promptVersion
schemaVersion
```

---

# 21. معماری فنی

## 21.1 Stack پیشنهادی

```txt
Frontend/Backend:
- Next.js App Router
- TypeScript
- TailwindCSS
- Server Components برای data fetching
- Client Components برای reader interactivity

Database:
- PostgreSQL
- Prisma یا Drizzle

Queue:
- Redis
- BullMQ

Storage:
- Local filesystem برای development
- MinIO یا S3-compatible storage برای production-like setup

Worker:
- Node.js worker جدا برای parse/translate

Infra:
- Docker
- Docker Compose
```

## 21.2 سرویس‌ها

```txt
web
 ├── Next.js app
 ├── Reader UI
 ├── Library UI
 ├── Settings UI
 └── API routes

worker
 ├── parse EPUB jobs
 ├── translation jobs
 ├── glossary jobs
 ├── summary jobs
 └── cleanup jobs

postgres
 └── persistent data

redis
 └── queue, job state, locks

storage
 ├── original EPUB
 ├── extracted assets
 └── future exports
```

## 21.3 Docker Compose پیشنهادی

```yaml
services:
  web:
    build:
      context: .
      dockerfile: Dockerfile
      target: web
    env_file:
      - .env
    ports:
      - "3000:3000"
    depends_on:
      - postgres
      - redis
      - storage

  worker:
    build:
      context: .
      dockerfile: Dockerfile
      target: worker
    env_file:
      - .env
    depends_on:
      - postgres
      - redis
      - storage

  postgres:
    image: postgres:latest
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
      POSTGRES_DB: smart_bilingual_reader
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:latest
    ports:
      - "6379:6379"

  storage:
    image: minio/minio
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: app
      MINIO_ROOT_PASSWORD: apppassword
    volumes:
      - minio_data:/data
    ports:
      - "9000:9000"
      - "9001:9001"

volumes:
  postgres_data:
  minio_data:
```

برای MVP می‌توان `storage` را حذف کرد و از local filesystem استفاده کرد، اما storage abstraction از اول باید وجود داشته باشد.

---

# 22. Data Model پیشنهادی

## 22.1 Book

```ts
type Book = {
  id: string;
  title: string;
  author?: string;
  language?: string;
  description?: string;
  publisher?: string;
  publishedAt?: Date;
  coverAssetId?: string;
  originalFileAssetId: string;
  processingStatus: BookProcessingStatus;
  processingError?: string;
  parserVersion: string;
  createdAt: Date;
  updatedAt: Date;
};
```

## 22.2 BookAsset

```ts
type BookAsset = {
  id: string;
  bookId: string;
  type:
    | "original_epub"
    | "cover"
    | "image"
    | "css"
    | "font"
    | "audio"
    | "video"
    | "other";
  originalPath: string;
  storedPath: string;
  mimeType?: string;
  sizeBytes?: number;
  hash?: string;
  createdAt: Date;
};
```

## 22.3 SpineItem

```ts
type SpineItem = {
  id: string;
  bookId: string;
  order: number;
  href: string;
  mediaType: string;
  title?: string;
  linear: boolean;
  rawHtml?: string;
  normalizedHtml?: string;
  plainText?: string;
};
```

## 22.4 TocItem

```ts
type TocItem = {
  id: string;
  bookId: string;
  parentId?: string;
  spineItemId?: string;
  order: number;
  title: string;
  href?: string;
  level: number;
};
```

## 22.5 ReadingUnit

```ts
type ReadingUnit = {
  id: string;
  bookId: string;
  spineItemId: string;
  order: number;
  titlePathJson: unknown;
  blockIdsJson: unknown;
  plainText: string;
  sourceHash: string;
  estimatedWordCount: number;
  charStart?: number;
  charEnd?: number;
  cfiStart?: string;
  cfiEnd?: string;
  createdAt: Date;
};
```

## 22.6 Block

```ts
type Block = {
  id: string;
  bookId: string;
  spineItemId: string;
  unitId: string;
  order: number;
  type: string;
  sourceText?: string;
  sourceHash?: string;
  structureJson: unknown;
  inlineMarksJson?: unknown;
  inlineSegmentsJson?: unknown;
  metadataJson?: unknown;
  originalHref: string;
  cfiStart?: string;
  cfiEnd?: string;
};
```

## 22.7 TranslationJob

```ts
type TranslationJob = {
  id: string;
  bookId: string;
  unitId: string;
  status: "queued" | "running" | "succeeded" | "failed" | "cancelled";
  priority: number;
  provider: string;
  model: string;
  targetLanguage: string;
  styleHash: string;
  glossaryHash: string;
  promptVersion: string;
  schemaVersion: string;
  errorCode?: string;
  errorMessage?: string;
  inputTokens?: number;
  outputTokens?: number;
  estimatedCost?: number;
  startedAt?: Date;
  finishedAt?: Date;
  createdAt: Date;
};
```

## 22.8 Translation

```ts
type Translation = {
  id: string;
  bookId: string;
  unitId: string;
  blockId: string;
  targetLanguage: string;
  provider: string;
  model: string;
  sourceHash: string;
  unitSourceHash: string;
  styleHash: string;
  glossaryHash: string;
  promptVersion: string;
  schemaVersion: string;
  parserVersion: string;
  translatedText?: string;
  translatedSegmentsJson?: unknown;
  translatedStructureJson?: unknown;
  status: "translated" | "cached" | "failed" | "stale" | "needs_review";
  version: number;
  errorMessage?: string;
  createdAt: Date;
  updatedAt: Date;
};
```

## 22.9 GlossaryTerm

```ts
type GlossaryTerm = {
  id: string;
  bookId: string;
  source: string;
  target: string;
  description?: string;
  status: "suggested" | "approved" | "rejected";
  confidence: "low" | "medium" | "high";
  createdBy: "ai" | "manual";
  firstSeenBlockId?: string;
  createdAt: Date;
  updatedAt: Date;
};
```

## 22.10 ReadingProgress

بدون user management، این progress در سطح workspace/book است.

```ts
type ReadingProgress = {
  id: string;
  bookId: string;
  currentUnitId?: string;
  currentBlockId?: string;
  scrollRatio?: number;
  updatedAt: Date;
};
```

## 22.11 Annotation

```ts
type Annotation = {
  id: string;
  bookId: string;
  unitId?: string;
  blockId?: string;
  type: "highlight" | "note" | "bookmark";
  selectedText?: string;
  note?: string;
  color?: string;
  cfiRange?: string;
  createdAt: Date;
  updatedAt: Date;
};
```

## 22.12 CostUsage

```ts
type CostUsage = {
  id: string;
  bookId?: string;
  jobId?: string;
  provider: string;
  model: string;
  inputTokens: number;
  outputTokens: number;
  estimatedCost?: number;
  createdAt: Date;
};
```

## 22.13 WorkspaceSettings

```ts
type WorkspaceSettings = {
  id: string;
  readerSettingsJson: unknown;
  aiSettingsJson: unknown;
  translationSettingsJson: unknown;
  costSettingsJson: unknown;
  updatedAt: Date;
};
```

---

# 23. API Design

## 23.1 Books

```txt
POST   /api/books/upload
GET    /api/books
GET    /api/books/:bookId
DELETE /api/books/:bookId
POST   /api/books/:bookId/reprocess
GET    /api/books/:bookId/processing-status
```

## 23.2 Reader

```txt
GET    /api/books/:bookId/toc
GET    /api/books/:bookId/units
GET    /api/books/:bookId/units/:unitId
PATCH  /api/books/:bookId/progress
```

## 23.3 Translation

```txt
POST   /api/books/:bookId/units/:unitId/translate
POST   /api/books/:bookId/units/:unitId/regenerate
POST   /api/books/:bookId/blocks/:blockId/regenerate
GET    /api/books/:bookId/translations
GET    /api/books/:bookId/units/:unitId/translations
```

## 23.4 Jobs

```txt
GET    /api/translation-jobs
GET    /api/translation-jobs/:jobId
POST   /api/translation-jobs/:jobId/retry
POST   /api/translation-jobs/:jobId/cancel
GET    /api/translation-events
```

## 23.5 Glossary

```txt
GET    /api/books/:bookId/glossary
POST   /api/books/:bookId/glossary
PATCH  /api/books/:bookId/glossary/:termId
DELETE /api/books/:bookId/glossary/:termId
POST   /api/books/:bookId/glossary/:termId/approve
POST   /api/books/:bookId/glossary/:termId/reject
POST   /api/books/:bookId/glossary/:termId/retroactive-update
```

## 23.6 Annotations

```txt
GET    /api/books/:bookId/annotations
POST   /api/books/:bookId/annotations
PATCH  /api/books/:bookId/annotations/:annotationId
DELETE /api/books/:bookId/annotations/:annotationId
```

## 23.7 Settings

```txt
GET    /api/settings
PATCH  /api/settings
```

## 23.8 Cost

```txt
GET    /api/cost/usage
GET    /api/books/:bookId/cost-estimate
GET    /api/books/:bookId/token-estimate
PATCH  /api/settings/cost
```

---

# 24. ساختار پروژه

```txt
src/
  app/
    page.tsx

    library/
      page.tsx

    books/
      [bookId]/
        page.tsx

    reader/
      [bookId]/
        page.tsx

    queue/
      page.tsx

    settings/
      page.tsx

    api/
      books/
      reader/
      translations/
      translation-jobs/
      glossary/
      annotations/
      settings/
      cost/

  components/
    library/
      BookCard.tsx
      UploadDropzone.tsx
      ProcessingBadge.tsx
      EmptyLibraryState.tsx

    reader/
      ReaderShell.tsx
      ReaderToolbar.tsx
      OriginalPane.tsx
      TranslationPane.tsx
      InterleavedPane.tsx
      SyncScrollProvider.tsx
      ProportionalScrollController.tsx
      TocSidebar.tsx
      FootnotePopover.tsx
      GlossaryTooltip.tsx
      TranslationStatusBadge.tsx
      TranslationSkeleton.tsx
      FocusModeOverlay.tsx
      BlockActionsMenu.tsx

    settings/
      ReaderSettingsForm.tsx
      AISettingsForm.tsx
      TranslationSettingsForm.tsx
      CostSettingsForm.tsx

    queue/
      TranslationJobTable.tsx
      JobStatusBadge.tsx

    glossary/
      GlossaryTable.tsx
      GlossarySuggestionCard.tsx
      RetroactiveUpdateDialog.tsx

    cost/
      CostDashboard.tsx
      TokenEstimateCard.tsx

  server/
    db/
      client.ts
      schema.ts

    storage/
      storageDriver.ts
      localStorageDriver.ts
      s3StorageDriver.ts

    epub/
      validateEpub.ts
      extractEpub.ts
      parsePackageDocument.ts
      parseManifest.ts
      parseSpine.ts
      parseNavigation.ts
      sanitizeXhtml.ts
      normalizeCss.ts
      normalizeXhtml.ts
      extractBlocks.ts
      extractInlineSegments.ts
      resolveFootnotes.ts
      resolveAssets.ts
      buildReadingUnits.ts

    translation/
      buildContextPack.ts
      buildTranslationPrompt.ts
      schemas.ts
      validateTranslationResponse.ts
      translationCache.ts
      translationMemory.ts
      translateUnit.ts
      estimateTokens.ts
      providers/
        openaiCompatibleProvider.ts
        anthropicProvider.ts
        openRouterProvider.ts
        localProvider.ts

    glossary/
      extractTerms.ts
      rankRelevantTerms.ts
      glossaryHash.ts
      retroactiveUpdate.ts

    cost/
      estimateBookCost.ts
      recordTokenUsage.ts
      enforceBudget.ts

    queue/
      queues.ts
      jobs.ts
      worker.ts
      processors/
        parseBookJob.ts
        translateUnitJob.ts
        summarizeChapterJob.ts
        extractGlossaryJob.ts
        retroactiveGlossaryUpdateJob.ts

  types/
    book.ts
    block.ts
    reader.ts
    translation.ts
    glossary.ts
    cost.ts
```

---

# 25. Settings و Configuration

## 25.1 env

```env
APP_NAME=Smart Bilingual Reader
APP_MODE=single_workspace

DATABASE_URL=postgresql://app:app@postgres:5432/smart_bilingual_reader
REDIS_URL=redis://redis:6379

STORAGE_DRIVER=local
STORAGE_LOCAL_PATH=/app/storage

S3_ENDPOINT=http://storage:9000
S3_BUCKET=smart-bilingual-reader
S3_ACCESS_KEY=app
S3_SECRET_KEY=apppassword
S3_FORCE_PATH_STYLE=true

DEFAULT_TARGET_LANGUAGE=fa

DEFAULT_AI_PROVIDER=openai_compatible
DEFAULT_AI_MODEL=
OPENAI_API_KEY=
ANTHROPIC_API_KEY=
OPENROUTER_API_KEY=
CUSTOM_AI_BASE_URL=

TRANSLATION_PROMPT_VERSION=1
TRANSLATION_SCHEMA_VERSION=1
EPUB_PARSER_VERSION=1

TRANSLATION_MAX_CONTEXT_TOKENS=12000
TRANSLATION_MAX_UNIT_WORDS=1200
TRANSLATION_MIN_UNIT_WORDS=700

TRANSLATION_CACHE_ENABLED=true
TRANSLATION_PREFETCH_ENABLED=true
TRANSLATION_PREFETCH_NEXT_UNITS=1

TRANSLATION_DAILY_TOKEN_LIMIT=300000
TRANSLATION_MAX_RETRIES=2
TRANSLATION_TIMEOUT_MS=90000

NEXT_PUBLIC_APP_NAME=Smart Bilingual Reader
```

## 25.2 Reader Settings

```ts
type ReaderSettings = {
  theme: "light" | "dark" | "sepia";
  originalFontFamily: string;
  translationFontFamily: string;
  fontSize: number;
  lineHeight: number;
  columnWidth: number;
  splitGap: number;
  panelOrder: "original_left" | "translation_left";
  syncScrollEnabled: boolean;
  proportionalScrollEnabled: boolean;
  focusModeEnabled: boolean;
};
```

## 25.3 AI Settings

```ts
type AISettings = {
  provider: string;
  model: string;
  baseUrl?: string;
  apiKey?: string;
  temperature: number;
  maxOutputTokens?: number;
  timeoutMs: number;
  retryCount: number;
  structuredOutputEnabled: boolean;
};
```

## 25.4 Translation Settings

```ts
type TranslationSettings = {
  targetLanguage: "fa";
  styleMode: TranslationStyleMode;
  preserveNames: boolean;
  preserveTerminology: boolean;
  preserveInlineMarks: boolean;
  translateCaptions: boolean;
  translateFootnotes: boolean;
  translateTables: boolean;
  prefetchEnabled: boolean;
  maxContextTokens: number;
};
```

## 25.5 Cost Settings

```ts
type CostSettings = {
  dailyTokenLimit: number;
  dailyCostLimit?: number;
  stopPrefetchAtBudgetPercent: number;
  warnAtBudgetPercent: number;
  maxConcurrentTranslationJobs: number;
};
```

---

# 26. Roadmap اجرایی

## Phase 0: Foundation

هدف: ساخت اسکلت پروژه.

خروجی‌ها:

- Next.js app.
- Database.
- Docker Compose.
- Storage abstraction.
- Queue setup.
- Basic settings.
- Library empty state.

---

## Phase 1: EPUB Core Pipeline

هدف: ورود کتاب به سیستم.

خروجی‌ها:

- Upload EPUB.
- Validate EPUB.
- Extract metadata.
- Extract cover.
- Parse manifest/spine.
- Parse TOC.
- Store assets.
- Book status management.
- Processing logs.

---

## Phase 2: Sanitization, Normalization, Blocks

هدف: تبدیل EPUB به ساختار قابل استفاده.

خروجی‌ها:

- XHTML sanitization.
- CSS normalization اولیه.
- DOM parsing.
- Block extraction.
- Inline segment extraction.
- Footnote mapping اولیه.
- Table extraction ساده.
- Image/caption mapping.
- ReadingUnit builder.

---

## Phase 3: Reader MVP

هدف: خواندن متن اصلی.

خروجی‌ها:

- Original reader.
- TOC sidebar.
- Reading progress.
- Reader settings.
- Split layout scaffold.
- Translation pane placeholder.
- Skeleton loading اولیه.

---

## Phase 4: Translation MVP

هدف: ترجمه current unit.

خروجی‌ها:

- Provider abstraction.
- Prompt builder.
- ContextPack اولیه.
- Structured response schema.
- Response validation.
- Translation persistence.
- Render translated pane.
- Regenerate current unit.
- Basic translation status.

---

## Phase 5: Queue, Cache, Prefetch

هدف: سریع‌تر و ارزان‌تر شدن.

خروجی‌ها:

- Unit cache.
- Job queue.
- Retry logic.
- Duplicate job prevention.
- Prefetch next unit.
- SSE status updates.
- Failed job retry.
- Queue dashboard.

---

## Phase 6: Glossary و Cost

هدف: کیفیت اصطلاحات و کنترل هزینه.

خروجی‌ها:

- Suggested glossary terms.
- Approve/reject/edit term.
- Glossary-aware translation.
- Stale translation status.
- Retroactive update اولیه.
- Token estimate.
- Cost dashboard.
- Budget limit.

---

## Phase 7: UX Polish

هدف: تجربه خواندن حرفه‌ای.

خروجی‌ها:

- Proportional sync scroll.
- Better skeleton loading.
- Cross-highlighting block-level.
- Focus mode.
- Footnote popover.
- Glossary tooltip.
- Block action menu.
- Interleaved view.

---

## Phase 8: Advanced Features

خارج از MVP:

- Sentence-level alignment.
- Ask about this page.
- Semantic search.
- pgvector.
- Book/chapter summaries.
- Export translated EPUB.
- Export bilingual PDF.
- Local models.
- Advanced table handling.
- Real pagination.

---

# 27. MVP دقیق

## MVP v1 باید داشته باشد

- Upload EPUB.
- Parse metadata.
- Extract cover.
- Extract TOC.
- Parse spine order.
- Sanitize XHTML.
- Extract heading/paragraph/list/image/caption/basic table.
- Build ReadingUnits.
- Original reader.
- Split view.
- Translation current unit.
- Context از previous/current/next unit.
- Structured JSON response.
- Validate response.
- Save translation.
- Render translated blocks.
- Unit-level cache.
- Basic provider/model/API key settings.
- Basic reader settings.
- Save one global progress per book.
- Docker Compose.
- Postgres.
- Redis worker.

## MVP v1 نباید داشته باشد

- user management.
- auth.
- permissions.
- payments.
- mobile app.
- PDF/OCR.
- DRM.
- export.
- full book auto translation.
- semantic search.
- vector database.
- sentence-level cross-highlighting.
- real browser pagination.
- collaboration.

---

# 28. معیارهای پذیرش

نسخه اول زمانی قابل قبول است که:

```txt
1. EPUB معتبر upload شود.
2. metadata و cover استخراج شود.
3. TOC نمایش داده شود.
4. ترتیب chapters مطابق spine باشد.
5. HTML خطرناک sanitize شود.
6. blockهای اصلی استخراج شوند.
7. ReadingUnitها بدون شکستن بی‌دلیل paragraph ساخته شوند.
8. original reader متن را درست نشان دهد.
9. translation pane ساختار مشابه داشته باشد.
10. current unit ترجمه شود.
11. context شامل previous/next unit excerpt باشد.
12. خروجی model اگر schema-invalid بود ذخیره نشود.
13. blockIdها در ترجمه حفظ شوند.
14. imageها در جای خود بمانند.
15. captionها قابل ترجمه باشند.
16. table ساده با cell mapping ترجمه شود.
17. cache برای request تکراری hit شود.
18. تغییر style باعث cache miss شود.
19. تغییر glossary approved باعث stale شدن ترجمه‌های وابسته شود.
20. sync scroll بر اساس blockId کار کند.
21. API key در client bundle قرار نگیرد.
22. failed jobs قابل retry باشند.
23. skeleton loading هنگام ترجمه نمایش داده شود.
24. daily token limit enforce شود.
```

---

# 29. تست، کیفیت و Observability

## 29.1 Golden EPUB Corpus

باید مجموعه‌ای از EPUBهای تست داشته باشیم:

- رمان ساده.
- کتاب non-fiction.
- کتاب با table.
- کتاب با footnote.
- کتاب با image/caption.
- کتاب با nested lists.
- کتاب با RTL/LTR mixed content.
- EPUB با HTML کثیف.
- EPUB با CSS عجیب.
- EPUB با chapterهای خیلی کوتاه.
- EPUB با paragraphهای خیلی بلند.

## 29.2 Unit Tests

- validate EPUB.
- parse manifest.
- parse spine.
- sanitize HTML.
- normalize CSS.
- extract blocks.
- extract inline segments.
- build ReadingUnits.
- build cache key.
- glossaryHash.
- validate translation response.

## 29.3 Integration Tests

- upload → parse → ready.
- unit translation → save → render.
- glossary change → translation stale.
- failed translation → retry.
- prefetch next unit.
- cost limit enforcement.

## 29.4 E2E Tests

- upload book.
- open reader.
- translate current unit.
- scroll sync.
- add glossary term.
- regenerate.
- highlight/note.
- check translation persisted after refresh.

## 29.5 Observability

ثبت شود:

- processing time per book.
- number of blocks.
- number of units.
- translation latency.
- cache hit rate.
- provider errors.
- validation failures.
- token usage.
- estimated cost.
- queue length.
- failed jobs.
- stale translations count.

---

# 30. ریسک‌ها و تصمیم‌های معماری

## 30.1 ریسک: HTMLهای کثیف ناشران

راه‌حل:

- sanitizer قوی.
- fallback parser.
- graceful degradation.
- حفظ raw original برای reprocess.

## 30.2 ریسک: خراب شدن ساختار توسط AI

راه‌حل:

- AI خروجی HTML ندهد.
- schema سخت‌گیرانه.
- blockId validation.
- segment/mark validation.
- fallback به block-by-block.

## 30.3 ریسک: هزینه بالا

راه‌حل:

- cache.
- prefetch محدود.
- budget.
- current unit first.
- no full book translation by default.
- token estimate.

## 30.4 ریسک: کندی ترجمه

راه‌حل:

- skeleton loading.
- prefetch next unit.
- SSE status.
- cache.
- queue priority.

## 30.5 ریسک: ناسازگاری glossary

راه‌حل:

- approved/suggested/rejected.
- glossaryHash.
- stale status.
- retroactive update.

## 30.6 ریسک: sync scroll ناپایدار

راه‌حل:

- blockId-based sync.
- proportional interpolation.
- anti-jerk lock.
- user toggle.

## 30.7 ریسک: ترجمه tableهای پیچیده

راه‌حل:

- cellId mapping.
- rowSpan/colSpan حفظ شود.
- tableهای خیلی بزرگ split یا skipped شوند.
- warning در UI.

---

# 31. نسخه نهایی خلاصه‌شده

Smart Bilingual Reader یک EPUB reader دوزبانه است که کتاب را از طریق یک pipeline ساختارمند پردازش می‌کند:

```txt
EPUB
  → Validate
  → Sanitize
  → Normalize
  → Extract Blocks
  → Build ReadingUnits
  → Build ContextPack
  → Translate with LLM
  → Validate JSON
  → Cache
  → Render synchronized bilingual reader
```

نکته اصلی این است که سیستم نباید روی Page بنا شود. Page در EPUB پایدار نیست. هسته محصول باید روی `ReadingUnit` و `Block` ساخته شود.

بهترین معماری این است:

```txt
App = parser + renderer + cache + queue + layout + state
AI  = translator + terminology assistant
```

ایده‌های محصولی مهم که محصول را حرفه‌ای می‌کنند:

- skeleton loading ساختاریافته.
- proportional sync scroll.
- focus mode.
- cross-highlighting.
- living glossary.
- retroactive glossary update.
- cost controller.
- queue priority.
- inline mark preservation بدون تولید HTML توسط AI.

MVP باید کوچک ولی درست باشد:

```txt
Upload EPUB
Parse and normalize
Build ReadingUnits
Render original
Translate current unit
Render translation
Cache
Queue next unit
Basic glossary
Basic cost control
```

بعد از پایدار شدن این هسته، می‌توان قابلیت‌های پیشرفته مثل sentence alignment، semantic search، ask-about-book، export EPUB/PDF و local models را اضافه کرد.
