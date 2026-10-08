# API_server — Binolla WebSocket Client

عميل WebSocket لمنصة **Binolla** يطبّق بروتوكول Socket.IO v4 / Engine.IO v4
مع دعم كامل للرسائل الثنائية (binary events بصيغة `451-[...]`).

## المميزات

- **تسجيل دخول HTTP تلقائي** عبر الإيميل/كلمة السر (بنفس بنية qx__1.py):
  - يحاكي Chrome عبر `CipherSuiteAdapter` (TLS fingerprinting)
  - يجرب عدة endpoints: `/api/auth/login`, `/api/login`, `/auth/login`, إلخ.
- **بدون أي أسئلة تفاعلية**: يقرأ `credentials.json` ويُسجّل الدخول تلقائياً
  إلى **حساب DEMO** افتراضياً.
- **يدعم النوعين** من رسائل Socket.IO:
  1. رسائل نصية: `42["event", data]`
  2. رسائل ثنائية: `451-["event", {_placeholder:true, num:0}]` + binary frame
- **يجلب جميع الأصول** المتوفرة في المنصة عبر `assets/list`.
- **يجلب نسبة الدفع** (payout %) لكل أصل عبر `asset/sentiment/subscribe`
  و`s_asset/sentiment/subscribe`.
- **يجلب السعر اللحظي** لكل أصل عبر `quotes/list`.
- **يشترك في الإشارات** (signals) لفريمات زمنية متعددة عبر `s_signals/asset/subscribe`.
- **يحفظ النتائج** في ملف JSON منظّم داخل `binolla_data/`.

## الاستخدام السريع

### 1) الإعداد الأول (مرة واحدة فقط)

انسخ `credentials.json.example` إلى `credentials.json` واملأه:

```json
{
  "email": "mineja5596@deertees.com",
  "password": "YOUR_PASSWORD",
  "is_demo": true
}
```

أو استخدم متغيرات البيئة:

```bash
export BINOLLA_EMAIL="mineja5596@deertees.com"
export BINOLLA_PASSWORD="YOUR_PASSWORD"
```

### 2) التشغيل

```bash
python binolla_api.py
```

سيقوم السكربت تلقائياً بـ:
1. تحميل `credentials.json`
2. تسجيل الدخول عبر HTTP (إن انتهت صلاحية JWT)
3. الاتصال بـ WebSocket إلى حساب DEMO
4. جلب كل الأصول + نسبة الدفع + الأسعار اللحظية
5. حفظ النتيجة في `binolla_data/assets_info_<timestamp>.json`

### 3) خيارات سطر الأوامر (اختيارية)

```bash
# تحديد أصل وعدد أيام لجلب الشموع أيضاً (إضافة لجلب الأصول)
python binolla_api.py --asset EURUSD_otc --days 7 --period 1

# استخدام حساب حقيقي بدلاً من DEMO
python binolla_api.py --real

# استخدام بروكسي
python binolla_api.py --proxy http://127.0.0.1:8080
```

## بنية رسائل WebSocket المدعومة

### رسائل نصية (Socket.IO v4 EVENT)
```
0{"sid":"...","pingInterval":25000,"pingTimeout":20000}   ← Engine.IO OPEN
40                                                        ← Socket.IO CONNECT (client→server)
40{"sid":"..."}                                            ← Socket.IO CONNECT_ACK (server→client)
42["authorization",{"token":"...","uaid":0,"userAccountType":1}]  ← JWT auth
42["s_authorization"]                                      ← auth confirmed
42["asset/list/change",[{"asset":"AUDCHF_otc","period":60}]]      ← change asset
42["asset/sentiment/subscribe","AUDCHF_otc"]               ← subscribe payout
42["s_asset/sentiment/subscribe"]                          ← subscribe global sentiment
42["s_signals/asset/subscribe",[{...}]]                    ← subscribe signals
42["quotes/list"]                                         ← request quotes
42["orders/opened/list"]                                  ← request open orders
2 / 3                                                     ← Engine.IO PING/PONG
```

### رسائل ثنائية (Socket.IO BINARY EVENT)
```
451-["s_assets/list",{"_placeholder":true,"num":0}]   ← header
<binary frame>                                          ← actual data
```

## الأحداث المُعالَجة

| Event | الوصف | التخزين |
|-------|-------|---------|
| `s_authorization` | تأكيد المصادقة | رفع async event |
| `s_assets/list` | قائمة كل الأصول | `api.assets_list` |
| `s_balances/list` | الأرصدة | `api.balances` |
| `s_settings/list` | الإعدادات | `api.settings` |
| `s_orders/opened/list` | الطلبات المفتوحة | `api.opened_orders` |
| `s_orders/closed/list` | الطلبات المغلقة | `api.closed_orders` |
| `s_history/last` | آخر شمعة | `api.history_last` |
| `s_history/region` | شموع تاريخية | `api.history_regions[index]` |
| `s_quotes/list` | الأسعار اللحظية | `api.assets_quotes[asset]` |
| `s_asset/sentiment` | **نسبة الدفع** | `api.assets_sentiment[asset]` |
| `s_signals/asset/change` | **إشارات الدفع** | `api.assets_signals[asset][tf]` |

## ملفات الإخراج

- `credentials.json` — التوكن + الإيميل/كلمة السر (يُحدّث تلقائياً)
- `binolla_data/assets_info_<timestamp>.json` — كل الأصول + نسبة الدفع + الأسعار
- `binolla_data/<asset>_<tf>m_<days>d_<rand>.json` — الشموع التاريخية (إن طُلبت)
- `binolla.log` — سجل التشغيل
- `ws_messages_<timestamp>.log` — سجل كل رسائل WebSocket

## المتطلبات

```bash
pip install requests websocket-client beautifulsoup4 certifi orjson
```
