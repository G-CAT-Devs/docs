---
title: "Architecture"
description: "معماری کلی GCAT — نسخه هدف (v3.1 merged)"
---

# معماری کلی GCAT — نسخه هدف (v3.1 merged)

تاریخ: ۱۴۰۵/۰۹/۰۱ · وضعیت: طرح / ایده‌پردازی (هیچ تغییری اجرا نشده)
مبنا: ممیزی واقعی سرور yotta-gcat + تصمیم‌های صریح بنیان‌گذار

## تصمیم‌های قطعی‌شده

- ❌ Swarm حذف (تک‌سرور، Compose کافی است)
- ❌ Open terminal کلاً خاموش (gateway + شبکه‌ها + volumeها)
- ❌ `code.gcat.ir` + سرویس T3 حذف
- ❌ `dsh.gcat.ir` + سرویس dsh روی هاست حذف
- ❌ `academia.gcat.ir` + کانتینر academia-core حذف (بعد از بکاپ داده)
- ❌ Panel به Frontend/Backend/DB شکسته نمی‌شود
- ✅ **Engine از Panel جدا می‌شود: دو ایمیج و دو کانتینر، با چرخه انتشار مستقل** (مرز جداسازی engine/product است نه FE/BE/DB)
- ✅ OmniRoute مستقیم از image رسمیِ نسخه‌پین‌شده اجرا می‌شود؛ `infra` فقط Compose، تنظیمات اجرا و روند ارتقای آن را نگه می‌دارد.
- 🔄 xray خاموش → 3x-ui جایگزین (بعد از اثبات معادل‌سازی پروکسی)
- 🔄 `ai.gcat.ir` → **`panel.gcat.ir`** (رابط کاربری پنل)
- 🆕 **`api.gcat.ir`** = تنها endpoint عمومی LLM API (خالص، بدون مسیر پنل)
- ✅ Grafana به‌عنوان سقف stack نظارتی لاگ‌محور (Loki + Alloy + Prometheus)
- ✅ Wiki = Docmost (احراز هویت محلی) · مستندات فنی = Fumadocs (static)
- ✅ CDN = ابرآروان · نام‌گذاری ریپو/ایمیج یکدست و اصولی (بخش ۳)
- ✅ یافته‌های امنیتی بسته `report/` پذیرفته شد: بستن aria2 :6888 · چرخش رازهای plaintext · فایروال nftables+DOCKER-USER · بکاپ از صفر (بخش ۱۰)

---

## ۱. نمای کلی

```
                         کاربران / کارکنان / API Clients
                                    │
                           ┌────────▼────────┐
                           │   ابرآروان CDN   │  DNS · TLS لبه · WAF · Cache
                           └────────┬────────┘
                                    │ HTTPS (گواهی معتبر origin)
                  ┌─────────────────▼──────────────────┐
                  │        yotta-gcat (Debian 13)      │
                  │      4 vCPU · RAM هدف: 16GB        │
                  │                                    │
                  │  nginx (هاست → پس از تثبیت: کانتینر)      │
                  │  ورودی وب عمومی :80/:443؛ SSH و VPN استثنا          │
                  │  کانفیگ ۱۰۰٪ از ریپوی infra          │
                  └───┬────────────────────────────┬───┘
                      │ 127.0.0.1:<port>           │
      ┌───────────────┼──────────────┬─────────────┼─────────────┐
      │               │              │             │             │
┌────▼─────┐   ┌─────▼─────┐  ┌─────▼─────┐ ┌─────▼─────┐ ┌─────▼─────┐
│omniroute │   │   panel   │  │   chat    │ │   wiki    │ │   drive   │
│ engine   │   │ panel+shop│  │  webui    │ │  docmost  │ │ cloudreve │
│ (image   │   │ (محصول    │  │ +postgres │ │ +postgres │ │ +pg+redis │
│  رسمی)   │   │  خودمان)  │  │ +redis    │ │ +redis    │ │           │
│ :20128   │   │  :3000    │  │  :8089    │ │  :8090    │ │  :5212    │
└──────────┘   └────┬──────┘  └───────────┘ └───────────┘ └───────────┘
      ▲              │ worker جدا
      │         ┌────▼──────┐  ┌───────────┐  ┌────────────┐  ┌──────────┐
      └─────────┤ shop-bot  │  │  hermes   │  │   3x-ui    │  │   docs   │
    panel→engine│ telegram  │  │ agent+ui  │  │ (جای xray) │  │ fumadocs │
    (internal)  │           │  │  :8787    │  │ پنل+پروکسی │  │  static  │
                └───────────┘  └───────────┘  └────────────┘  └──────────┘

┌────────────────────── نظارت (apps/observability) ──────────────────────┐
│ Alloy (docker logs + journald + nginx) ──push──► Loki                  │
│ node-exporter + cAdvisor + blackbox ◄──pull── Prometheus               │
│                    Grafana :3001 (داشبورد+هشدار از Git)                │
│                          │ هشدار + لینک Runbook → تلگرام               │
└────────────────────────────────────────────────────────────────────────┘

سرورهای دیگر (هلنوس و آینده): فقط Alloy + node-exporter → push به همین stack

حذف‌شده‌ها: Swarm · open-terminal · dsh · academia-core · T3/code · xray (پس از جایگزینی)
```

<Card title="نکته فنی: جداسازی Engine/Panel" icon="cube">
امروز روی سرور **یک کانتینر** (`omniroute-panel`) هر دو پروسه را با supervisord اجرا می‌کند. جداسازی به دو کانتینر یعنی:
- آپدیت engine از upstream بدون ری‌بیلد پنل
- دیپلوی و rollback مستقل
- محدودیت مصرف منابع جدا

هزینه‌اش تقریباً صفر است (همان دو پروسه، فقط در دو کانتینر). این جداسازی سربار شبکه، عملیات، آزمون سازگاری و مدیریت داده دارد که باید اندازه‌گیری شود.
</Card>

---

## ۲. نقشه دامنه → سرویس → پورت (نسخه هدف)

| دامنه | سرویس | آپ‌استریم | احراز هویت |
|---|---|---|---|
| gcat.ir / www | لندینگ/پنل (default_server) | 127.0.0.1:3000 | — |
| **panel.gcat.ir** 🔄 (جای ai) | پنل + فروشگاه | 127.0.0.1:3000 | پنل (SSO) |
| auth.gcat.ir / llm | alias پنل (برای تداوم SSO) | 127.0.0.1:3000 | پنل |
| **api.gcat.ir** 🆕 | **تنها endpoint عمومی LLM API** — خالص /v1/* | 127.0.0.1:20128 | API key |
| chat.gcat.ir / chat.g-cat.ir | Open WebUI | 127.0.0.1:8089 | OIDC → پنل |
| drive.gcat.ir | Cloudreve | 127.0.0.1:5212 | خودش |
| wiki.gcat.ir 🆕 | Docmost | 127.0.0.1:8090 | ایمیل/رمز + basic-auth |
| docs.gcat.ir 🆕 | Fumadocs static (فایل روی دیسک، سرو مستقیم nginx) | /opt/stacks/docs/out | عمومی یا basic-auth |
| grafana.gcat.ir 🆕 | Grafana | 127.0.0.1:3001 | لاگین + basic-auth |
| xui.gcat.ir 🆕 (نام موقت) | 3x-ui | 127.0.0.1:2053 | لاگین + basic-auth + مسیر مخفی |
| hermes-webui.gcat.ir | hermes | 127.0.0.1:8787 | خودش |
| ~~ai.gcat.ir~~ | → panel.gcat.ir (ریدایرکت 301 موقت برای کلاینت‌های قدیمی API → api) | — | — |
| ~~oinawofb*.gcat.ir~~ | ❌ بازنشسته — داشبورد engine فقط داخلی (پیشنهاد: engine.gcat.ir با basic-auth، یا فقط از طریق پنل) | — | — |
| ~~code / dsh / academia~~ | ❌ حذف کامل (سرویس + vhost + DNS + داده بعد از بکاپ) | — | — |

### قواعد ثابت nginx

- همه vhostها: `http2 on` + گواهی wildcard معتبر Let's Encrypt (DNS-01) + snippet مشترک proxy/websocket
- `api.gcat.ir` هیچ مسیر پنل/داشبورد ندارد؛ فقط API خالص + rate limit سبک در nginx
- استاتیک Next هرگز از snapshot هاست سرو نمی‌شود → همیشه پروکسی به خود اپ (درس حادثه 404)
- real-IP ابرآروان · برای وب فقط nginx پورت عمومی · DB/Redis بدون پورت publish

⚠️ **وابستگی‌های تغییر دامنه** (قبل از سوییچ باید همگی آماده شوند):

1. رکوردهای DNS جدید در ابرآروان: panel, api (+ حذف code/dsh/academia/oinawofb)
2. تنظیمات OIDC چت (issuer/redirectها الان auth.gcat.ir هستند) + envهای SSO پنل
3. کلاینت‌های API موجود که به ai.gcat.ir وصل‌اند → اطلاع‌رسانی + ریدایرکت/alias موقت
4. گواهی wildcard جدید باید قبل از سوییچ صادر شده باشد

---

## ۳. ریپوها و نام‌گذاری (org: G-CAT-Devs)

### قرارداد نام‌گذاری

- ریپو: kebab-case، کوتاه، بدون پیشوند تکراری (org خودش namespace است)
- ایمیج: `ghcr.io/g-cat-devs/<repo>:<git-sha>` — نام ایمیج = نام ریپو؛ هرگز `latest` در production
- پروژه Compose = نام ریپو · کانتینرها: `<project>-<service>` (مثلاً `chat-db`)
- سرویس‌های third-party (docmost, cloudreve, 3x-ui, grafana...) ریپو نمی‌خواهند؛ compose آن‌ها در `infra/apps/` با نسخه pin شده

### ریپوهای هدف

| ریپو / مسیر | نوع | محتوا | ایمیج |
|---|---|---|---|
| `infra/apps/omniroute/` | استقرار و تنظیمات اجرا در monorepo `infra`؛ بدون fork و build اختصاصی | Compose، `.env.example`، healthcheck و runbook ارتقا | `diegosouzapw/omniroute:<version>` |
| **`panel`** 🆕 | محصول | پنل + فروشگاه وب + بات تلگرام (از `omni/panel` استخراج می‌شود)؛ یک ایمیج، دو نقش: web + bot worker | `ghcr.io/g-cat-devs/panel:<sha>` |
| **`chat`** 🔄 (تغییر نام `chat-gcat`) | fork | Open WebUI + سفارشی‌سازی‌ها + compose (pg/redis) | `ghcr.io/g-cat-devs/chat:<sha>` |
| **`infra`** 🆕 | زیرساخت | nginx + compose همه stackهای third-party + اسکریپت‌ها + داشبوردها/آلارم‌ها | — (nginx بعداً ایمیج خودش) |
| **`docs`** 🆕 | مستندات | Fumadocs: معماری، ADR، runbook فنی، API reference | static export → nginx روی سرور (بدون ایمیج، ~صفر RAM) |

### سرنوشت ریپوهای موجود

| ریپوی فعلی | اقدام |
|---|---|
| `omni` | پس از استخراج `panel` و تأیید استقرار image رسمی OmniRoute → **archive** (تاریخچه حفظ می‌شود) |
| `chat-gcat` | **rename → `chat`** (GitHub ریدایرکت خودکار دارد) + پاک‌سازی برنچ‌ها + CI جدید |
| `academic-chat-assistant` / `gcat-academia` / `Academia-backup` | archive بعد از بکاپ داده‌های academia |
| `open-terminal` / ریپوهای ترمینال | archive |
| `Chat-Design-System`, `ai-lms`, ... | خارج از این معماری؛ دست‌نخورده |

### نحوهٔ اجرای OmniRoute

- تصمیم بنیان‌گذار: **هیچ تغییری در کد engine نمی‌دهیم.** پس fork و image واسط ارزشی ایجاد نمی‌کنند.
- Compose در `infra/apps/omniroute/` مستقیماً image رسمیِ `diegosouzapw/omniroute:<version>` را اجرا می‌کند. تگ باید دقیقاً پین شود؛ `latest` ممنوع است.
- **ارتقای upstream = یک PR که نسخه image را در Compose عوض می‌کند** → pull → تست دود → دیپلوی.
- مرز قرارداد با پنل فقط HTTP است (آدرس قابل دسترسی از شبکهٔ پنل، مثلاً `http://omniroute-engine:20128` روی شبکهٔ مشترک محدود؛ loopback داخل کانتینر به host یا کانتینر دیگر اشاره نمی‌کند).
- اگر روزی نیاز به patch پیدا شد، آن‌وقت fork می‌کنیم — نه زودتر. (قانون: fork آخرین راه‌حل است، نه اولین.)
- نسخهٔ اجراشده در inventory و پیام deploy ثبت می‌شود تا rollback با تغییر تگ به نسخهٔ قبلی انجام شود.

### ساختار infra

```
infra/
├── nginx/
│   ├── nginx.conf
│   ├── snippets/          # proxy-common, tls, arvan-realip, websocket, api-ratelimit
│   └── sites/             # panel, api, chat, wiki, docs, drive,
│                          # grafana, xui, hermes, redirects
├── apps/                     # هر stack یک compose.yaml مستقل
│   ├── wiki/              # docmost + pg + redis
│   ├── omniroute/         # omniroute
│   ├── drive/             # cloudreve + pg + redis (از وضعیت فعلی منتقل)
│   ├── observability/     # grafana + loki + prometheus + alloy + exporters
│   │   └── grafana/       # dashboards + alerting — provisioned از Git
│   └── xui/               # 3x-ui
├── firewall/              # ufw rules + DOCKER-USER + rollback timer
├── hosts/
│   └── yotta-gcat.yaml    # inventory: سرویس‌ها، پورت‌ها، health URLها
├── scripts/               # validate, deploy-nginx, smoke-test, backup, cleanup
├── .github/workflows/     # validate (nginx -t) + deploy (SSH → reload → smoke → rollback)
└── docs/                  # RUNBOOK.md, BOOTSTRAP.md, POSTMORTEM.md, PORTS.md، ADRها
```

> **Secretها** (.env، کلید TLS، توکن بات) فقط روی سرور یا GitHub Secrets — هرگز در Git.

### ساختار داخلی ریپوهای جدید

<Card title="panel (محصول: پنل + فروشگاه + بات)" icon="monitor-cog">
```
panel/
├── .github/workflows/
│   ├── ci.yml               # lint + test + build
│   └── release.yml          # یک ایمیج → ghcr.io/g-cat-devs/panel:<sha>
├── app/                     # Next.js: پنل + shop (کد فعلی omni/panel)
├── bot/                     # worker بات تلگرام (همان ایمیج، command متفاوت)
├── docker/
│   ├── Dockerfile
│   └── entrypoint.sh        # نقش web یا bot از env
├── deploy/
│   └── compose.production.yaml   # panel-web :3000 + panel-bot (بدون پورت)
├── migrations/              # schema دیتابیس (SQLite /data)
├── .env.example
└── README.md
```
</Card>

<Card title="chat (rename از chat-gcat)" icon="bot-message-square">
```
chat/
├── .github/workflows/       # ci + release → ghcr.io/g-cat-devs/chat:<sha>
├── (سورس Open WebUI fork)
├── docker/Dockerfile
├── deploy/
│   └── compose.production.yaml   # chat-webui :8089 + chat-db + chat-redis
├── .env.example
└── README.md
```
</Card>

<Card title="docs (Fumadocs)" icon="book-open">
```
docs/
├── .github/workflows/deploy.yml  # build (output: export) → rsync out/ به سرور
├── app/                     # صفحات Fumadocs
├── content/                 # MDX: معماری، ADR، API reference
├── next.config.ts           # output: 'export'
└── README.md
```
</Card>

### ۳.۱ نقشه کامل پورت / کانتینر / مسیر روی سرور

قرارداد فاز nginx میزبان (جدول زیر وضعیت هدف این فاز است، نه معماری شبکه نهایی): همه فایل‌ها زیر `/opt/stacks/<stack>/` · همه publishها فقط `127.0.0.1` · DB/Redis بدون publish · نام کانتینر `<stack>-<service>`.

| stack (پروژه compose) | کانتینر | ایمیج | پورت host (فقط loopback) | پورت داخلی | مسیر روی سرور |
|---|---|---|---|---|---|
| **omniroute** | `omniroute-engine` | diegosouzapw/omniroute:`<version>` | 127.0.0.1:**20128** | 20128 | `/opt/stacks/infra/apps/omniroute/` |
| **panel** | `panel-web` | ghcr.io/g-cat-devs/panel:\<sha\> | 127.0.0.1:**3000** | 3000 | `/opt/stacks/panel/` |
| **panel** | `panel-bot` | همان ایمیج (command: bot) | — | — | `/opt/stacks/panel/` |
| **chat** | `chat-webui` | ghcr.io/g-cat-devs/chat:\<sha\> | 127.0.0.1:**8089** | 8080 | `/opt/stacks/chat/` |
| **chat** | `chat-db` | postgres:16-alpine (pin) | — | 5432 | `/opt/stacks/chat/` |
| **chat** | `chat-redis` | redis:7-alpine (pin) | — | 6379 | `/opt/stacks/chat/` |
| **wiki** | `wiki-app` | docmost/docmost:\<pin\> | 127.0.0.1:**8090** | 3000 | `/opt/stacks/wiki/` |
| **wiki** | `wiki-db` | postgres:16-alpine | — | 5432 | `/opt/stacks/wiki/` |
| **wiki** | `wiki-redis` | redis:7-alpine | — | 6379 | `/opt/stacks/wiki/` |
| **drive** | `drive-app` | cloudreve/cloudreve:\<pin\> | 127.0.0.1:**5212** | 5212 | `/opt/stacks/drive/` |
| **drive** | `drive-db` | postgres:16-alpine | — | 5432 | `/opt/stacks/drive/` |
| **drive** | `drive-redis` | redis:7-alpine | — | 6379 | `/opt/stacks/drive/` |
| **drive** | (aria2 داخل app) | — | 127.0.0.1:**6888** ❌ عمومی نباشد | 6888 | — |
| **xui** | `xui-app` | ghcr.io/mhsanaei/3x-ui:\<pin\> | 127.0.0.1:**2053** (پنل) | 2053 | `/opt/stacks/xui/` |
| **xui** | (inboundهای پروکسی) | — | **عمومی فقط پورت‌های VPN** (ufw allow صریح) | — | — |
| **observability** | `obs-grafana` | grafana/grafana:\<pin\> | 127.0.0.1:**3001** | 3000 | `/opt/stacks/observability/` |
| **observability** | `obs-loki` | grafana/loki:\<pin\> | 127.0.0.1:**3100** | 3100 | `/opt/stacks/observability/` |
| **observability** | `obs-prometheus` | prom/prometheus:\<pin\> | 127.0.0.1:**9090** | 9090 | `/opt/stacks/observability/` |
| **observability** | `obs-alloy` | grafana/alloy:\<pin\> | — | — | `/opt/stacks/observability/` |
| **observability** | `obs-node-exporter` | prom/node-exporter:\<pin\> | 127.0.0.1:**9100** | 9100 | — |
| **observability** | `obs-cadvisor` | gcr.io/cadvisor/cadvisor:\<pin\> | 127.0.0.1:**8081** | 8080 | — |
| **observability** | `obs-blackbox` | prom/blackbox-exporter:\<pin\> | 127.0.0.1:**9115** | 9115 | — |
| **docs** | — (static) | بدون کانتینر | nginx مستقیم | — | `/opt/stacks/docs/out/` |
| **hermes** (میزبان، فعلاً) | systemd | — | 127.0.0.1:**8787** (webui) · **9119** (dashboard) | — | `/opt/hermes/` (مستند در inventory) |
| **nginx** (edge) | میزبان → کانتینر پس از تثبیت | nginx:1.28-alpine | **0.0.0.0:80/443** (ورودی عمومی وب) | — | `/opt/stacks/infra/nginx/` |

**پورت‌های عمومی مجاز**: **22 (SSH، فقط کلید + IP مدیریت) · 80 · 443 · inboundهای VPN 3x-ui (صریح در ufw)**. سایر پورت‌های ضروری در فاز میزبان فقط loopback و DB/Redis بدون publish هستند.

داده‌های persistent: `/opt/stacks/<stack>/data/` (bind mount) یا volume نام‌دار `<stack>_<service>_data` — هر stack فقط داده خودش. بکاپ‌ها: `/opt/backups/`.

---

## ۴. جریان دیپلوی

```
push به main هر اپ (panel / chat)
  → Actions: تست → build → ghcr.io/g-cat-devs/<repo>:<sha>
  → SSH: /opt/stacks/<repo>/compose.yaml (از ریپو) + .env (روی سرور)
  → docker compose pull && up -d   (فقط همان پروژه)
  → health-check → موفق: ثبت sha / شکست: rollback به sha قبلی

تغییر `infra/apps/omniroute/compose.yaml`
  → Actions: compose config → SSH: pull image رسمی → smoke test → rollback با تگ قبلی
```

- تغییر nginx: فقط از infra → `nginx -t` → **reload** → smoke-test همه دامنه‌ها
- دیپلوی engine نباید پنل را restart کند و برعکس (مزیت اصلی جداسازی)
- Bootstrap سرور جدید = clone infra + BOOTSTRAP.md + restore بکاپ‌ها

---

## ۵. داده و بکاپ

| داده | محل | بکاپ |
|---|---|---|
| engine + panel (SQLite) | /data | dump روزانه + خارج سرور |
| chat / wiki / drive (Postgres) | volume داکر | pg_dump روزانه + خارج سرور |
| فایل‌های کاربران | volumeها | Restic به storage خارج سرور، با آزمون سازگاری |
| کانفیگ‌ها و secrets | Git و مخزن امن جداگانه | نسخه بازیابی خارج سرور؛ Git جای بکاپ secrets نیست |
| داده academia (قبل از حذف) | — | بکاپ کامل + تأیید restore، بعد حذف |

### ۵.۱ بکاپ دوره‌ای خارج سرور

- **مقصد اصلی:** object storage سازگار با S3 خارج از سرور، با Restic رمزنگاری‌شده، incremental و retention صریح؛ ارائه‌دهنده، ظرفیت، هزینه، سیاست نگه‌داری و مسئول نگهداری هنوز نیازمند تصمیم‌اند. رمز repository و اطلاعات دسترسی خارج از Git و نسخه بازیابی آن‌ها خارج از سرور و قابل دسترس برای افراد مجاز نگهداری شود؛ بدون کلید بازیابی ممکن نیست.
- روزانه در زمان کم‌ترافیک: برای هر PostgreSQL از `pg_dump -Fc` و برای SQLite از API سازگار مانند `sqlite3 .backup` استفاده و صحت خروجی بررسی شود؛ فایل‌های پایدار و تنظیمات لازم نیز با Restic پوشش داده شوند. هماهنگی سازگاری داده و فایل‌های مرتبط و وضعیت WAL و شیوه snapshot هر سرویس پیش از اجرا مشخص شود؛ Git جای بکاپ داده و secrets نیست.
- تلگرام برای **اعلان نتیجه** است؛ کپی اضطراری رمزنگاری‌شده فقط اختیاری و ثانویه، پس از بررسی محدودیت‌های جاری API، اندازه و retention است؛ هرگز جایگزین S3 یا معیار موفقیت بکاپ اصلی نیست.
- هدف پیشنهادی، **نه تعهد تأییدشده**: RPO حداکثر ۲۴ ساعت و RTO حداکثر ۴ ساعت؛ نیازمند تأیید کسب‌وکار و اثبات با اندازه‌گیری. شکست dump، آپلود یا verification هشدار فوری دارد.
- ماهانه از S3 در محیط مجزا بازیابی و صحت دیتابیس، فایل‌ها، کلیدها و زمان بازیابی را ثبت کنید؛ بازگردانی اضطراری قبل از تغییرات حیاتی نیز باید آزموده شود. retention محلی در صورت استفاده تابع فضای دیسک و سیاست مصوب است.

### ۵.۲ SQLite و هم‌زمانی

داده SQLite پنل روی volume پایدار با مجوز حداقلی باشد؛ backup با روش سازگار، نه کپی خام فایل در حال نوشتن. پیش از اجرای web و bot با دسترسی هم‌زمان، الگوی تراکنش و بار واقعی، WAL در صورت پشتیبانی، timeout و lock contention آزموده شوند. در صورت ناسازگاری، تک‌نویسنده یا مهاجرت برنامه‌ریزی‌شده به PostgreSQL بررسی شود؛ volume اشتراکی به‌تنهایی هم‌زمانی ایمن را تضمین نمی‌کند.

<Card title="بکاپ تلگرام (پیاده‌سازی جزئی از v3)" icon="send" href="#backup-telegram">
<details>
<summary>جزئیات کامل پیاده‌سازی بکاپ تلگرام (از v3 حفظ شده)</summary>

**اسکریپت `infra/scripts/backup-to-telegram.sh` با systemd timer روزانه (ساعت کم‌ترافیک، مثلاً ۰۴:۰۰):**

1. **Dump هر دیتابیس:**
   - Postgresها: `pg_dump -Fc` داخل کانتینر (`chat-postgres`, `cloudreve-db`, `wiki-db` بعداً)
   - SQLiteها (engine/panel): `sqlite3 .backup` (نه کپی خام فایل — برای consistency)

2. **فشرده‌سازی + رمزنگاری:** هر dump → `zstd` → `age` با کلید عمومی (کلید خصوصی فقط روی لپ‌تاپ‌های بنیان‌گذاران، هرگز روی سرور). دلیل: دیتابیس چت/فروشگاه حاوی داده کاربر و توکن است؛ کانال تلگرام نباید نسخه plaintext داشته باشد.

3. **ارسال به تلگرام:** فایل رمزنگاری‌شده به کانال/گروه خصوصی «GCAT Backups» با بات ops؛ کپشن: نام سرویس + تاریخ + حجم + sha256.

4. **نوتیف نتیجه:** خلاصه به گروه ops (موفق ✅ با حجم‌ها / شکست 🔴 فوری). شکست ارسال = alert، نه سکوت.

5. **نگه‌داری محلی:** ۷ نسخه آخر در `/opt/backups/` (برای restore سریع)؛ پاکسازی خودکار قدیمی‌ها.

6. **تست restore ماهانه:** دانلود آخرین فایل از تلگرام → رمزگشایی → restore در کانتینر موقت → گزارش به ops.

**محدودیت‌های تلگرام که طراحی باید رعایت کند:**
- سقف آپلود بات: **۵۰MB per file**. اگر dump بزرگ‌تر شد → split به قطعات ۴۵MB با پسوند شماره‌دار + اسکریپت reassemble در restore.
- تلگرام **مقصد ثانویه** است نه یگانه: اگر روزی حجم از سقف گذشت یا بات محدود شد، مقصد اصلی object storage (S3 سازگار) جایگزین/مکمل می‌شود — اسکریپت از اول دو-مقصدی نوشته می‌شود (`TARGETS=telegram,s3`).
- پیام‌های بکاپ در topic جدا از alertها تا نویز نگیرد.

> **قانون:** بکاپی که restore آن تست نشده، بکاپ نیست. ماهی یک‌بار تست restore.
</details>
</Card>

---

## ۶. نظارت، هشدار و نوتیف تلگرام

```
هشدار (Grafana Alerting / health-check / backup) → بات تلگرام → گروه ops
        │ هر پیام: severity + سرویس + خلاصه + لینک Grafana + لینک Runbook (Docmost)
        ▼
grafana.gcat.ir → لاگ همان سرویس/بازه (Loki) → Runbook → حل → ثبت درس
```

### ۶.۱ کانال نوتیف تلگرام (مصوب)

| جزء | تصمیم |
|---|---|
| بات | **بات مستقل ops** (`gcat_ops_bot`) — جدا از بات فروشگاه؛ توکن فقط در `.env` روی سرور و GitHub Secrets |
| مقصد | گروه خصوصی «GCAT Ops» (هر سه هم‌بنیان‌گذار) + topic جدا برای alerts/backup/deploy |
| مسیر ارسال | Grafana contact point (webhook → relay سبک) + اسکریپت‌های backup/deploy مستقیم به Bot API |
| فرمت پیام | `🔴/🟡/🟢 [severity] service@host — خلاصه — لینک داشبورد — لینک runbook` |
| ضدنویز | group_wait=30s · group_interval=5m · repeat_interval=4h · silence در پنجره دیپلوی |
| پیام‌های مثبت | deploy موفق (sha جدید) · backup موفق روزانه (خلاصه حجم) · resolve شدن alert |

### ۶.۲ رویدادهایی که حتماً نوتیف می‌شوند

1. **Alertها:** دامنه non-200 بیش از ۲ دقیقه · swap > 500MB · دیسک > 85٪ · RAM در دسترس < 1GB · restart کانتینر · SSL < 14 روز · healthcheck شکست‌خورده
2. **Backup:** موفق/شکست هر اجرای pg_dump/restic + نتیجه تست restore ماهانه (شکست backup = 🔴 فوری)
3. **Deploy:** شروع/موفق/شکست + rollback خودکار هر اپ و nginx (با sha قبل/بعد)
4. **امنیتی:** ورود SSH جدید (fail2ban ban) · تغییر کانفیگ nginx خارج از pipeline (drift)

**Runbookهای اولیه:** 404 استاتیک · فشار swap · گواهی پشت ابرآروان · restart هر سرویس · سوییچ دامنه API.
هر incident بعد از حل → ۱۰ دقیقه نوشتن runbook در Docmost.

---

## ۷. بودجه منابع (هدف: RAM 16GB)

| گروه | مصرف تقریبی |
|---|---|
| omniroute (heap سقف 2GB) | ~2GB |
| panel + shop-bot | ~0.5GB |
| chat stack | ~1.6GB |
| wiki stack 🆕 | ~0.9GB |
| drive | ~0.25GB |
| hermes | ~0.35GB |
| 3x-ui (جای xray) | ~0.15GB |
| observability 🆕 | ~0.9GB |
| OS + nginx + headroom | ~1.2GB |
| **جمع** | **~7.9GB** |

> این اعداد صرفاً فرض اولیه‌اند، نه اندازه‌گیری یا ظرفیت تضمین‌شده. پیش از استقرار کامل، در بار واقعی و دوره نماینده مصرف حافظه و CPU کانتینرها، disk I/O، restart count، latency درخواست HTTP و اتصال‌های دیتابیس را ثبت و headroom تعیین کنید. Loki باید retention و سقف عملی ذخیره‌سازی با هشدار ظرفیت و بررسی رشد داده داشته باشد؛ retention به‌تنهایی سقف سخت حجم نیست. Prometheus نیز retention و ظرفیت دیسک مشخص داشته باشد. استقرار تدریجی و بازبینی پس از اندازه‌گیری الزامی است.

**آزادسازی‌های مصوب**: Swarm · terminal · dsh (~220MB) · academia (~100MB) · xray (پس از جایگزینی) · heap از 4GB→2GB · ~۱۰GB آرشیو مایگریشن + کانتینرهای exited.
حذف dsh/academia/terminal بخشی از بار جدید را جبران می‌کند، ولی بدون RAM هدف 16GB، افزودن سرویس‌های جدید توصیه نمی‌شود.

---

## ۸. اولویت و مراحل اجرا

هیچ مرحله‌ای مجوز اجرای خودکار نیست؛ شرایط و تأییدهای بخش ۱۱ برقرارند. اقدامات امنیتی فوری را در اولین فرصت امن و پس از بکاپ اضطراری انجام دهید؛ در خطر فعال، مهار فوری با تأیید انسانی ممکن است جلوتر از تکمیل بکاپ لازم شود.

| ترتیب | اقدام و دروازه |
|---|---|
| ۱ | بکاپ اضطراری خارج سرور و تست restore داده‌های حیاتی پیش از تغییر مخرب |
| ۲ | بستن پورت‌های ناامن، چرخش secrets افشاشده و فایروال سازگار، با rollback و تست مستقل |
| ۳ | رفع nginx، TLS، HTTP/2 و static؛ ابتدا nginx روی host بماند |
| ۴ | ساخت infra و deploy کانفیگ host nginx از Git، اعتبارسنجی و حذف drift دستی |
| ۵ | جداسازی Engine و Panel با تست سازگاری و بکاپ |
| ۶ | GHCR و CI/CD مستقل با health check و rollback هر stack |
| ۷ | تغییر دامنه‌ها و OIDC با طرح بازگشت و سازگاری کلاینت‌ها |
| ۸ | Wiki و مستندات پس از آماده بودن منابع |
| ۹ | اندازه‌گیری و استقرار تدریجی observability با retention و بودجه دیسک |
| ۱۰ | پس از تثبیت، انتقال nginx به Docker با شبکه و TLS آزموده‌شده و rollback |

### شبکه و استقلال stackها

تا وقتی nginx روی host است، فقط upstreamهای لازم روی `127.0.0.1` منتشر می‌شوند؛ DB و Redis publish ندارند. پس از containerize شدن nginx، هدف اتصال edge و سرویس‌های وب به شبکه مشترک proxy و اتصال هر برنامه به شبکه داخلی اختصاصی خودش است؛ پورت اپ‌ها به‌طور پیش‌فرض publish نمی‌شود. شبکه Docker به‌تنهایی تضمین عدم دسترسی host یا جداسازی کامل نیست؛ دسترسی host و سطح دسترسی اعضای شبکه باید جداگانه کنترل و آزموده شود. ارتباط پنل با engine نیز نیازمند شبکه مشترک محدود و آدرس قابل resolve است؛ استثناهای host و VPN باید مستند و بازبینی شوند. DNS کانتینری، renew گواهی، دسترسی static و rollback قبل از cutover آزموده شوند.

هر stack فایل Compose و project name مستقل دارد؛ فرمان deploy باید صریحاً فایل و project همان stack را انتخاب کند. هیچ `docker compose up -d` سراسری در ریشه infra مجاز نیست. تغییر nginx یا monitoring نباید OmniRoute را restart کند؛ health check، نسخه، تست و rollback مستقل‌اند و CI بر اساس مسیر تغییر کرده و وابستگی‌های صریح انتخاب می‌شود.

---

## ۹. ریسک‌ها و سؤال‌های باز

1. **3x-ui جای xray:** اثبات معادل‌سازی پروکسی برای مصرف‌کننده‌های فعلی (احتمالاً engine روی socks 10808) قبل از خاموشی xray.
2. **سوییچ ai→panel/api:** کلاینت‌های API موجود نباید بشکنند — alias/ریدایرکت موقت + اطلاع‌رسانی.
3. **OIDC:** تغییر دامنه پنل یعنی به‌روزرسانی هم‌زمان envهای چت و پنل؛ سوییچ باید اتمیک باشد.
4. **استخراج panel از omni:** باید تاریخچه Git فایل‌های panel حفظ شود (git filter-repo) و مرز engine/panel در کد دقیق مشخص شود.
5. **RAM:** افزودن سرویس‌های جدید بدون RAM هدف 16GB و ارزیابی بار انجام نشود.
6. **Quora translator:** هنوز تعریف نشده — خارج از معماری.
7. **داشبورد engine:** oinawofb* بازنشسته شود؟ دسترسی ادمین به داشبورد engine از کجا باشد (پیشنهاد: engine.gcat.ir با basic-auth) — نیاز به تأیید.
8. **auth.gcat.ir:** به‌عنوان alias پنل نگه داشته شود (تداوم SSO) یا آن هم به panel منتقل شود — نیاز به تأیید.

---

## ۱۰. ادغام بسته `report/` (نسخه ۰.۱ — ممیزی موازی)

بسته `F:\gcat\report\` یک ممیزی موازی با ۱۸ مشکل شناسه‌دار (P-01…P-18) است. یافته‌هایش پذیرفته شد با این نگاشت:

| یافته report | سرنوشت در این معماری |
|---|---|
| P-01/P-02/P-03 (ایمیج ID-only، اجرای دستی، نبود CI) | ✅ حل با ریپوهای جدید + GHCR + تگ sha (بخش ۳ و ۴) |
| P-04 (رازهای plaintext در compose.json و تاربال‌ها) | ✅ اولویت ۲: انتقال امن secrets و **چرخش همه رازهای افشاشده** (WEBUI_SECRET_KEY، OAUTH_CLIENT_SECRET، GCAT_INTERNAL_API_SECRET، پسوردهای DB). چرخش کلید WebUI = خروج کاربران → در پنجره کم‌ترافیک با اعلام قبلی |
| P-05 (aria2 RPC روی 0.0.0.0:6888 بدون auth) | ✅ اولویت ۱: bind به 127.0.0.1 (Cloudreve فقط loopback لازم دارد) |
| P-06 (نبود فایروال؛ داکر ufw را دور می‌زند) | ✅ اولویت ۲: سیاست default deny و مجوزهای صریح SSH، وب و VPN؛ UFW به‌تنهایی برای publish داکر کافی نیست. backend واقعی Docker و firewall ابتدا شناسایی شود؛ DOCKER-USER فقط در backend و مسیری که پشتیبانی می‌شود، با قواعد persistent و تست‌شده استفاده شود. قبل از اعمال، rollback زمان‌بندی‌شده فعال و SSH و سرویس‌های عمومی از session مستقل تست شوند؛ فقط پس از موفقیت rollback لغو شود. fail2ban و SSH کلیدی با حفظ دسترسی مدیریتی اعمال شوند |
| P-07 (کانتینرهای root، بدون cap_drop) | ✅ در composeهای جدید: non-root در حد امکان + cap_drop |
| P-08 (نبود بکاپ — crontab خالی) | ✅ اولویت ۱: بکاپ اضطراری و restore؛ سپس: pg_dump + restic به object storage خارج سرور + تست restore ماهانه. ادعای «بکاپ تلگرام ۳ساعته» در context تأیید نشد |
| P-09/P-10 (فشار RAM/دیسک) | ✅ هم‌راستا: heap ↓، پاکسازی ~۱۰GB + ۶GB dangling، هدف RAM 16GB |
| P-11 (omniroute-panel unhealthy) | ⚠️ ناپایدار است نه حل‌شده: گزارش ساعت ۱۸:۱۰ unhealthy دید، ممیزی ۲۱:۴۵ healthy — کانتینر ۲۱:۰۳ recreate شده بود. علت ریشه‌ای باید بررسی شود؛ جداسازی دو کانتینر به‌تنهایی رفع آن را تضمین نمی‌کند |
| P-12/P-14 (drift، compose.json غیراستاندارد) | ✅ اولویت ۱: per-stack compose.yaml در `/opt/stacks/<repo>/` + drift check |
| P-13 (host network) | ✅ در بازسازی omniroute/panel حذف می‌شود؛ publish فقط 127.0.0.1 |
| P-15 (دو مدل استقرار) | ✅ با تصمیم‌های جدید: dsh/code/academia حذف، xray→3x-ui کانتینری، hermes فعلاً systemd + مستند در hosts inventory |
| P-16 (upstream شکسته + TLS) | ✅ code حذف؛ TLS معتبر DNS-01 در اولویت ۱ |
| P-17 (نبود observability) | ✅ ارتقا یافته: به‌جای Uptime Kuma، stack کامل Grafana+Loki+Prometheus (تصمیم بنیان‌گذار) |
| P-18 (زوائد) | ✅ اولویت ۱/۲ با تأیید + grep وابستگی قبل از حذف |

**تصمیم‌های باز گزارش که با تصمیم‌های جدید جایگزین شدند**: D-01 (nginx میزبان→کانتینر پس از تثبیت ✅)، D-02 (registry محلی→**GHCR**)، D-05 (compose ✅ Swarm حذف)، D-07 (Kuma→**Grafana stack**)، D-10 (dsh حذف/xray جایگزینی)، Q6 (code→حذف کامل)، Q12 (کامل).

**سؤال‌های گزارش که همچنان بازند**: Q2 (پنجره downtime)، Q8 (مقصد بکاپ: object storage کدام provider؟)، Q9 (دسترسی SSH چند نفر)، R3 (دامنه‌های g-cat.ir بمانند؟)، R7 (والیوم chat1_migrated کی بازنشسته شود؟).

---

## ۱۱. سیاست اجرای Agent (پیشنهادی؛ نیازمند تأیید انسانی)

<Warning>
این سند فعلاً طرح است و به‌خودی‌خود اختیار دسترسی یا اجرا نمی‌دهد. تا تصویب صریح این سیاست، برای هر تغییر عملیاتی تأیید موردی لازم است.
</Warning>

- **مستقل و فقط در محیط مجاز:** خواندن اسناد و پیکربندی غیرحساس، تحلیل، تدوین طرح، ساخت و اجرای تست محلی غیرمخرب؛ بدون افشای secret در لاگ یا خروجی.
- **نیازمند تأیید انسانی پیش از اجرا:** هر تغییر production، deployment و rollback، تغییر firewall/SSH/DNS/TLS، چرخش secret، مهاجرت داده، حذف یا archive، تغییر backup و restore و هر اقدام دارای downtime یا هزینه. طرح باید هدف، دامنه، پیش‌نیاز، اثر، روش راستی‌آزمایی و بازگشت را مشخص کند؛ تأیید کلی سند جای تأیید عملیات نیست.
- **ممنوع:** درج یا انتشار secret، حذف داده بدون بکاپ و restore تأییدشده، بازکردن عمومی DB، دورزدن تأیید یا کنترل امنیتی، deploy سراسری همه stackها، ادعای موفقیت بدون آزمون، و تغییر خارج از دامنه مجاز. در رخداد امنیتی، استثنای اضطراری تنها با دستور صریح انسان و ثبت آن ممکن است.
- پیش از هر اقدام خطرناک، وضعیت واقعی، دسترسی مستقل و امکان rollback بررسی و خروجی عملیات بدون داده حساس ثبت شود؛ در ابهام یا شکست توقف و گزارش شود.

---

## ۱۲. ADRها

تصمیم‌های مستقل و پیامدهایشان در `docs/adr/` نگهداری می‌شوند؛ این سند نمای کلی است. در صورت تعارض، ابتدا اختلاف با بازبینی و تأیید انسانی رفع شود، نه اجرای خودکار.

---

<Card title="مخزن مستندات" icon="github" href="https://github.com/G-CAT-Devs/docs">
این معماری در مخزن <code>G-CAT-Devs/docs</code> نگهداری می‌شود. تمام ADRها، runbookها و مستندات فنی در اینجا.version کنترل می‌شوند.
</Card>