# Inbox Triage Assistant

### Creator : [selim@agentron.ai](selim@agentron.ai)

Bu agentflow, Gmail inbox’taki uygun email’leri alip AI ile triage eden ve sonucu native `Collections` icinde log olarak kaydeden bir akisidir. V1 tasarimi customer-facing cevap yazmaz, Gmail label basmaz ve email modify etmez; yalnizca classification ve logging yapar.

---

## Overview

Bu template kurumsal ekipler icin sade ve guvenli bir inbox triage copilot mantigiyla tasarlanmistir:

- Gmail inbox’tan secilen query’ye uyan son email’i alir
- Gerekli konfigrasyon eksikse setup guidance reply dondurur
- Email’i normalize eder ve email fingerprint uretir
- Aynı email’in daha once loglanip loglanmadigini `Collections` uzerinden kontrol eder
- Yeni email ise tek bir `LLM` node’u ile triage eder
- Sonucu native `Collections` icinde triage log kaydi olarak acar
- Kullaniciya `recordId`, kategori, priority ve onerilen owner bilgisini dondurur

Bu tasarimda otomatik yonlendirme, mail gonderimi veya Gmail label uygulama yoktur. V1 amaci, mailbox’tan gelen is yukunu standart, arastirilabilir bir log yapisina cevirmektir.

---

## Flow Mimarisi

```text
Start Button
   |
   v
Validate Required Config
   |
   +--> Config missing
   |       |
   |       v
   |   Setup Guidance Reply
   |
   +--> Config OK
           |
           v
      List Gmail Messages
           |
           v
      Prepare Email Context
           |
           +--> No eligible email
           |       |
           |       v
           |   No Email Reply
           |
           +--> Email found
                   |
                   v
              Build Email Fingerprint
                   |
                   v
              Search Existing Triage Logs
                   |
                   v
              Detect Existing Record
                   |
                   +--> Already processed
                   |       |
                   |       v
                   |   Already Processed Reply
                   |
                   +--> New email
                           |
                           v
                      AI Triage LLM
                           |
                           v
                      Build Log Payload
                           |
                           v
                      Create Triage Record
                           |
                           v
                      Success Reply
```

---

## Node Gruplari

| Grup | Node Tipi | Amac |
|------|-----------|------|
| **Inbox Triage Assistant** | Start | Inbox-driven triage calismasini baslatir ve runtime state'i yukler. |
| **Validate Required Config** | CustomFunction | `triageCollectionId` ve `inboxQuery` alanlarini kontrol eder. |
| **Config Missing?** | Condition | Eksik setup varsa guidance branch'ine gecer. |
| **Setup Guidance Reply** | DirectReply | Import sonrasi doldurulmasi gereken ayarlari bildirir. |
| **List Gmail Messages** | Gmail | Inbox query’ye gore email listeler. |
| **Prepare Email Context** | CustomFunction | Ilk email’i secip gonderen, konu ve govde bilgisini normalize eder. |
| **No Eligible Email?** | Condition | Uygun email yoksa akis no-email reply ile biter. |
| **No Email Reply** | DirectReply | Query’ye uyan email bulunmadigini bildirir. |
| **Build Email Fingerprint** | CustomFunction | Email iceriginden deterministik dedup fingerprint’i uretir. |
| **Search Existing Triage Logs** | Collections | `emailFingerprint` field’i uzerinden mevcut kaydi arar. |
| **Detect Existing Record** | CustomFunction | Search sonucundan `alreadyProcessed` state’ini uretir. |
| **Already Processed?** | Condition | Ayni email daha once loglanmis mi diye karar verir. |
| **Already Processed Reply** | DirectReply | Tekrar isleme yerine mevcut fingerprint’in daha once kaydedildigini bildirir. |
| **AI Triage LLM** | LLM | Email’i kategori, priority, summary, reason ve suggested owner alanlarina ayirir. |
| **Build Log Payload** | CustomFunction | `recordId`, `createdAtIso` ve `triageRecordPayload` uretir. |
| **Create Triage Record** | Collections | Collections icinde triage log kaydi acar. |
| **Success Reply** | DirectReply | Kayit tamamlandigini, record ID ve triage sonucunu bildirir. |

---

## Start ve Flow State

Start node `startButton` modunda calisir. Form kullanilmaz; template tamamen inbox-driven tasarlanmistir.

Start state icinde tanimli key’ler:

- `inboxQuery`
  Default: `in:inbox is:unread -in:sent -in:drafts -in:trash`
- `triageCollectionId`
- `defaultStatus`
  Default: `triaged`
- `defaultOwnerTeam`
  Default: `ops`
- `recordPrefix`
  Default: `TRI`
- `maxMessages`
  Default: `1`
- `recordId`
- `createdAtIso`
- `sourceFromEmail`
- `sourceFromName`
- `sourceSubject`
- `sourceBody`
- `emailFingerprint`
- `hasRequiredConfig`
- `setupGuidance`
- `hasEligibleEmail`
- `alreadyProcessed`
- `triageCategory`
- `triagePriority`
- `triageSummary`
- `triageReason`
- `triageSuggestedOwner`
- `triageRecordPayload`

### State Akisi

1. `Validate Required Config` node’u `hasRequiredConfig` ve `setupGuidance` uretir.
2. `Prepare Email Context` node’u email’den gelen alanlari state’e yazar.
3. `Build Email Fingerprint` node’u dedup icin kullanilan fingerprint’i uretir.
4. `Detect Existing Record` node’u `alreadyProcessed` state’ini doldurur.
5. `AI Triage LLM` node’u classification alanlarini state’e yazar.
6. `Build Log Payload` node’u final Collections payload’ini hazirlar.

---

## Gmail Intake Davranisi

`List Gmail Messages` node’u:

- `gmailType = messages`
- `messageAction = listMessages`
- `messageQuery = {{ $flow.state.inboxQuery }}`
- `messageMaxResults = {{ $flow.state.maxMessages }}`

`Prepare Email Context` node’u:

- Ilk email’i secer
- `sourceFromEmail`, `sourceFromName`, `sourceSubject`, `sourceBody` alanlarini doldurur
- Email yoksa `hasEligibleEmail = false`
- Email varsa `hasEligibleEmail = true`

### Onemli Teknik Not

Bu repodaki mevcut agentflow Gmail `listMessages` implementasyonu:

- normalize edilmis `from`, `subject`, `body` gibi alanlari dondurur
- ancak `messageId` ve `threadId` alanlarini expose etmez
- ayrica Gmail `modify/label/thread` aksiyonlari agentflow backend’de aktif degildir

Bu nedenle bu template:

- Gmail label basmaz
- Gmail status degistirmez
- dedup icin Gmail ID kullanmaz
- dedup’i `emailFingerprint` + `Collections` search uzerinden yapar

---

## Dedup Stratejisi

`Build Email Fingerprint` node’u:

- `from + subject + normalized body` uzerinden deterministik bir fingerprint uretir
- fingerprint formatini `fp_<hash>` olarak tutar

`Search Existing Triage Logs` node’u:

- `mode = searchRecords`
- `collectionId = {{ $flow.state.triageCollectionId }}`
- `query = {{ $flow.state.emailFingerprint }}`
- `searchFields = ["emailFingerprint"]`
- `limit = 1`

`Detect Existing Record` node’u:

- search sonucunda kayit varsa `alreadyProcessed = true`
- yoksa `alreadyProcessed = false`

Bu yapi sayesinde ayni email tekrar triage edilmez. Gmail uzerinde label basmaya gerek kalmadan dedup korunur.

---

## AI Triage Davranisi

Tek bir `LLM` node’u email’i triage eder.

### Structured Output Fields

- `category`
  Enum: `support`, `sales`, `finance`, `internal`, `spam`
- `priority`
  Enum: `low`, `normal`, `high`, `critical`
- `summary`
- `reason`
- `suggestedOwner`
  Enum: `support`, `sales`, `finance`, `ops`, `leadership`

### Prompt Kurallari

- Bilgi uydurma yok
- Email kisa ve operasyonel sekilde siniflandirilir
- `spam` ise bu durum net olarak belirtilir
- `support` ve `finance` gibi aksiyon gerektiren emaillerde priority makul sekilde belirlenir
- Cikti customer-facing degil, tamamen internal ops icindir

---

## Collections Log Creation

`Build Log Payload` node’u su alanlari finalize eder:

- `recordId`
  Format: `<recordPrefix>-<YYYYMMDD>-<short id>`
- `createdAtIso`
- `triageRecordPayload`

### Required Collection

Tek collection varsayilir:

- `triageCollectionId`

### Minimum Schema

`triageCollectionId` ile hedeflenen collection minimum su field’lari icermelidir:

- `recordId`
- `createdAt`
- `status`
- `fromEmail`
- `fromName`
- `subject`
- `messageBody`
- `emailFingerprint`
- `category`
- `priority`
- `summary`
- `reason`
- `suggestedOwner`

### Payload Icerigi

Collections kaydina yazilan minimum veri:

- `recordId`
- `createdAt`
- `status`
- `fromEmail`
- `fromName`
- `subject`
- `messageBody`
- `emailFingerprint`
- `category`
- `priority`
- `summary`
- `reason`
- `suggestedOwner`

---

## User-Facing Replies

### Setup Guidance Reply

Eksik config durumunda su maddeleri ozellikle soylenir:

- `triageCollectionId`
- `inboxQuery`
- Gmail credential setup
- LLM model secimi

### No Email Reply

Query’ye uyan email yoksa akis durur ve bilgi reply’i dondurur.

### Already Processed Reply

Ayni fingerprint daha once loglanmissa akis yeni kayit acmaz ve duplicate oldugunu bildirir.

### Success Reply

Basarili durumda:

- `recordId`
- `category`
- `priority`
- `suggested owner`

bilgileri dondurulur.

---

## Kurulum

### Gereksinimler

- [EmploidAI](https://app.emploid.ai)
- En az bir LLM provider baglantisi
- Native Collections erisimi
- Gmail OAuth credential

### Import Adimlari

1. EmploidAI’a giris yapin.
2. **Agentflows** ekranina gidin.
3. Import islemini baslatin.
4. `Inbox Triage Assistant Agent Flow.json` dosyasini import edin.
5. `AI Triage LLM` node’unda model secin.
6. `Start` node icindeki `triageCollectionId` degerini doldurun.
7. `inboxQuery` degerini kendi mailbox ihtiyaciniza gore guncelleyin.
8. `Gmail` node’una `gmailOAuth2` credential baglayin.
9. Flow’u tek email senaryosunda test edin.

---

## Collections Pre-setup

Bu template tek bir collection bekler.

### Onerilen Collection

Collection adi ornek:

- `inboxTriageLogs`

### Minimum Field Listesi

- `recordId`
- `createdAt`
- `status`
- `fromEmail`
- `fromName`
- `subject`
- `messageBody`
- `emailFingerprint`
- `category`
- `priority`
- `summary`
- `reason`
- `suggestedOwner`

### Field Tipi Onerileri

- `recordId`, `fromEmail`, `fromName`, `subject`, `category`, `priority`, `suggestedOwner`, `emailFingerprint` icin string/text
- `messageBody`, `summary`, `reason` icin long text
- `createdAt` icin datetime
- `status` icin string/select benzeri alan

### Neden Gmail Label Degil Collections Log?

Bu repo’daki mevcut agentflow Gmail node sinirlari nedeniyle:

- label uygulama
- thread modify
- message modify

davranislari v1’de güvenilir agentflow capability olarak kullanilmiyor. Bu nedenle en stabil kurumsal tasarim, triage sonucunu `Collections` icinde saklamaktir.

---

## Kullanim Notlari

- Bu template tek run’da tek email isler.
- Batch processing v1 disindadir.
- Gmail modify/label aksiyonu yoktur.
- Dedup `emailFingerprint` uzerinden yapilir.
- Ayni email tekrar query’ye takilsa bile, daha once loglandiysa tekrar kayit acilmaz.
- Kategori seti bilerek sade tutulmustur:
  `support`, `sales`, `finance`, `internal`, `spam`
- Varsayilan owner team `ops` olarak gelir; LLM uygun durumlarda farkli owner onerir.

---

## Onerilen Modeller

- `gpt-4o`
- `claude-sonnet-4-6`

Bu template classification odakli oldugu icin orta-ust kalite bir model yeterlidir. Daha hizli ama daha zayif modeller kritik finance veya spam ayriminda daha dengesiz davranabilir.

---

## Test Senaryolari

### JSON / wiring

- JSON parse edilmeli
- Tum node ve edge referanslari gecerli olmali
- CustomFunction node’larinin JavaScript’i compile edilmeli
- State variable referanslari hatasiz olmali

### Functional scenarios

1. `triageCollectionId` bos
   Setup guidance branch’i calismali
2. `inboxQuery` bos
   Setup guidance branch’i calismali
3. Query’ye uyan email yok
   No-email branch’i calismali
4. Email bulundu
   Fingerprint uretilmeli
5. Ayni fingerprint mevcut
   Already-processed branch’i calismali
6. Yeni email
   LLM `category`, `priority`, `summary`, `reason`, `suggestedOwner` dondurmali
7. Collections log creation
   Yeni triage log kaydi olusmali
8. Success reply
   `recordId`, `category`, `priority`, `suggested owner` bilgileri dondurulmali

---

## Lisans

MIT License
