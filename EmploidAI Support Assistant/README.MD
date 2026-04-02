# EmploidAI Support Assistant

### Creator : [selim@agentron.ai](selim@agentron.ai)

Bu agentflow, form tabanli bir destek talebi toplama ve ic ticketing otomasyonu akisidir. Akis; support formunu toplar, gerekli konfigrasyonu kontrol eder, destek mesajini tek bir `LLM` node'u ile triage eder, sonucu native `Collections` icinde ticket kaydi olarak olusturur ve ic ekibe `Gmail` uzerinden bildirim gonderir.

---

## Overview

Bu tasarim v1 kapsaminda yalnizca ticket intake ve internal operations otomasyonuna odaklanir:

- Kullanici bir support formu doldurur
- Akis once gerekli `startState` ayarlarinin eksik olup olmadigini kontrol eder
- Konfigrasyon eksikse mutasyon yapmadan setup guidance reply dondurur
- Konfigrasyon tam ise destek talebi tek bir `LLM` node'u ile normalize edilir
- `Collections` icinde standart bir ticket kaydi olusturulur
- `high` ve `critical` talepler farkli bir Gmail branch'ine yonlendirilir
- Son kullaniciya ticket ID, kategori, priority ve owner team bilgisi dondurulur

Bu template; customer-facing cevap yazmak yerine, destek operasyon ekibinin ilk ticket yakalama ve siniflandirma isini hizlandirmak icin tasarlanmistir.

---

## Flow Mimarisi

```text
Start Support Form
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
      AI Triage LLM
           |
           v
      Build Ticket Payload
           |
           v
      Create Ticket Record
           |
           v
      Priority Condition
         |        |
         |        +--> Urgent Notification Email
         |                 |
         |                 v
         |            Success Reply
         |
         +-----------> Standard Notification Email
                           |
                           v
                      Success Reply
```

---

## Node Gruplari

| Grup | Node Tipi | Amac |
|------|-----------|------|
| **Support Intake Form** | Start | Destek talebini form ile toplar ve runtime state'i baslatir. |
| **Validate Required Config** | CustomFunction | `supportCollectionId` ve `notificationEmail` ayarlarini kontrol eder. |
| **Config Missing?** | Condition | Eksik setup varsa guidance reply branch'ine gecis yapar. |
| **Setup Guidance Reply** | DirectReply | Eksik ayarlari ve import sonrasi yapilmasi gerekenleri kullaniciya bildirir. |
| **AI Triage LLM** | LLM | Talebi kategori, priority, summary, suggested team ve next action alanlarina normalize eder. |
| **Build Ticket Payload** | CustomFunction | `ticketId`, `createdAtIso`, record payload ve notification metinlerini olusturur. |
| **Create Ticket Record** | Collections | Pre-configured collection ID icinde yeni support ticket kaydi acar. |
| **Urgent Priority?** | Condition | `high` veya `critical` triage sonucunu ayri Gmail branch'ine yollar. |
| **Urgent Notification Email** | Gmail | Yuksek oncelikli ticket'lari `urgentNotificationEmail` veya fallback olarak `notificationEmail` adresine yollar. |
| **Standard Notification Email** | Gmail | Normal ticket'lari `notificationEmail` adresine yollar. |
| **Success Reply** | DirectReply | Ticket olustugunu, hangi kategoriye dustugunu ve hangi ekibe atandigini bildirir. |

---

## Formda Toplanan Alanlar

Start node `formInput` modunda calisir ve asagidaki alanlari toplar:

- `requesterName`
- `requesterEmail`
- `companyName`
- `subject`
- `message`
- `urgency`
  Allowed values: `low`, `normal`, `high`, `critical`
- `referenceId`
- `productArea`
- `consentToContact`
  Boolean
- `preferredChannel`
  Allowed values: `email`, `phone`, `no_preference`

Bu contract README ile JSON arasinda birebir aynidir. Template'i degistirirken field isimlerini bozarsaniz downstream node'larda variable resolution kirilir.

---

## AI Triage Davranisi

Tek bir `LLM` node'u support icerigini structured output olarak normalize eder.

### Structured Output Fields

- `category`
  Enum: `bug`, `billing`, `access`, `feature_request`, `general_question`, `account`, `integration`, `other`
- `priority`
  Enum: `low`, `normal`, `high`, `critical`
- `summary`
  Kisa ve operasyonel ic ticket ozeti
- `suggestedTeam`
  Enum: `support`, `product`, `engineering`, `billing`, `ops`
- `nextAction`
  Ekibin atmasi gereken en yararli ilk adim

### Prompt Kurallari

- Bilgi uydurma yok
- Kisa ve operasyonel ozet
- Formdaki `urgency` ile kullanici mesaji birlikte degerlendirilir
- Musteriye otomatik cevap yazilmaz
- V1 yalnizca internal ticket enrichment yapar

---

## Collections Veri Modeli

Bu template native `Collections` uzerinden calisir. V1 kapsaminda template kendi data model'ini yaratmaz; bunun yerine import oncesi veya sonrasi sizin manuel olarak olusturdugunuz mevcut bir collection ID'yi kullanir.

`Create Ticket Record` node'u asagidaki minimum payload ile record olusturur:

- `ticketId`
- `createdAt`
- `status`
- `requesterName`
- `requesterEmail`
- `companyName`
- `subject`
- `message`
- `urgency`
- `category`
- `priority`
- `summary`
- `assignedTeam`
- `nextAction`
- `referenceId`
- `productArea`
- `preferredChannel`
- `consentToContact`

### Alanlarin Amaci

- `ticketId`: Dis gorunen benzersiz ticket numarasi. Format `SUP-YYYYMMDD-XXXXXX`
- `createdAt`: Kaydin ISO timestamp'i
- `status`: Varsayilan olarak `new`
- `requesterName`: Talebi acan kisi
- `requesterEmail`: Talebi acan e-posta adresi
- `companyName`: Talebin bagli oldugu sirket
- `subject`: Kisa konu satiri
- `message`: Orijinal destek mesaji
- `urgency`: Formda kullanicinin sectigi oncelik
- `category`: AI triage sonucu uretilen kategori
- `priority`: AI triage sonucu uretilen operasyonel priority
- `summary`: Internal queue icin kisa ozet
- `assignedTeam`: Ilk yonlendirme icin onerilen ekip
- `nextAction`: En uygun ilk aksiyon
- `referenceId`: Kullanici tarafindan saglanan referans numarasi
- `productArea`: Ilgili urun veya modul bilgisi
- `preferredChannel`: Tercih edilen iletisim kanali
- `consentToContact`: Geri donus izni

---

## Flow State

Start node'unda asagidaki state key'leri tanimlidir:

- `supportCollectionId`
- `notificationEmail`
- `urgentNotificationEmail`
- `defaultStatus`
  Default: `new`
- `defaultAssigneeTeam`
  Default: `support`
- `ticketPrefix`
  Default: `SUP`
- `ticketId`
- `createdAtIso`
- `triageCategory`
- `triagePriority`
- `triageSummary`
- `triageSuggestedTeam`
- `triageNextAction`
- `ticketRecordPayload`
- `notificationSubject`
- `notificationBody`

Template'in ic calismasi icin ek olarak su runtime helper key'leri de vardir:

- `hasRequiredConfig`
- `setupGuidance`
- `notificationTargetEmail`
- `isUrgent`

### State Akisi

1. `Validate Required Config` node'u `hasRequiredConfig` ve `setupGuidance` uretir.
2. `AI Triage LLM` node'u `triageCategory`, `triagePriority`, `triageSummary`, `triageSuggestedTeam`, `triageNextAction` alanlarini doldurur.
3. `Build Ticket Payload` node'u `ticketId`, `createdAtIso`, `ticketRecordPayload`, `notificationSubject`, `notificationBody`, `notificationTargetEmail`, `isUrgent` alanlarini doldurur.
4. `Collections` ve `Gmail` node'lari bu state degerlerini kullanarak record ve bildirim olusturur.

---

## Notification Davranisi

Bu template iki Gmail notification davranisi kullanir:

- `priority = high` veya `priority = critical`
  Flow `Urgent Notification Email` branch'ine gecer.
- Diger tum durumlar
  Flow `Standard Notification Email` branch'ine gecer.

### Routing Mantigi

- Standard branch:
  `notificationEmail`
- Urgent branch:
  `urgentNotificationEmail`
  Eger bos ise fallback olarak `notificationEmail`

### Email Icerigi

Her iki bildirim maili de su alanlari icerir:

- Ticket ID
- Created At
- Priority
- Category
- Assigned Team
- Requester identity
- Company
- Form urgency
- Reference ID
- Product Area
- Summary
- Next Action
- Original Message

Bu bildirim customer-facing degildir. Yalnizca ic operasyon ekibine gider.

---

## Kurulum

### Gereksinimler

- [EmploidAI](https://app.emploid.ai)
- En az bir LLM provider baglantisi
- Native Collections erisimi
- Gmail OAuth credential

### Import Adimlari

1. EmploidAI'a giris yapin.
2. **Agentflows** ekranina gidin.
3. Import islemini baslatin.
4. `EmploidAI Support Assistant Agent Flow.json` dosyasini import edin.
5. `AI Triage LLM` node'unda model secimini yapin.
6. Her iki `Gmail` node'unda `gmailOAuth2` credential baglayin.
7. `Start` node icindeki `startState` degerlerini kendi ortaminiza gore guncelleyin.
8. Test icin formu doldurup akisi calistirin.

---

## Collections Pre-setup

Bu adim onemlidir. Akis import edildiginde `supportCollectionId` otomatik dolu gelmez.

### Onerilen Kurulum

1. EmploidAI icinde mevcut bir data model secin veya yeni bir data model olusturun.
2. Bu data model altinda `supportTickets` benzeri bir collection olusturun.
3. Asagidaki field'lari ekleyin:

- `ticketId`
- `createdAt`
- `status`
- `requesterName`
- `requesterEmail`
- `companyName`
- `subject`
- `message`
- `urgency`
- `category`
- `priority`
- `summary`
- `assignedTeam`
- `nextAction`
- `referenceId`
- `productArea`
- `preferredChannel`
- `consentToContact`

### Field Tipi Onerileri

- Metin alanlari icin text/string
- `createdAt` icin date/datetime
- `consentToContact` icin boolean
- `message` ve `summary` icin long text destekleniyorsa long text

### Start State'e Yazilacak Degerler

`Start` node icinde su alanlari doldurun:

- `supportCollectionId`
  Collections icindeki hedef collection ID
- `notificationEmail`
  Standart ic bildirim mailbox'i
- `urgentNotificationEmail`
  Opsiyonel acil bildirim mailbox'i
- `defaultStatus`
  Ornek: `new`
- `defaultAssigneeTeam`
  Ornek: `support`
- `ticketPrefix`
  Ornek: `SUP`

### Neden README ile Pre-setup?

V1 kapsaminda template'in amaci ticketing core akisini gostermektir. Collection olusturma, field migration ve schema provisioning adimlari kuruma gore degisebildigi icin template bunu otomatiklestirmez.

---

## Gmail Kurulumu

Template iki ayri Gmail node'u icerir:

- `Urgent Notification Email`
- `Standard Notification Email`

Her iki node icin de import sonrasi:

1. `gmailOAuth2` credential secin
2. Gerekirse test maili gonderin
3. Kurumunuzun mailbox politikasina gore alici adreslerini `startState` uzerinden duzenleyin

### Dikkat Edilecek Nokta

Config eksiginde flow mail gondermeye calismaz. Ilk condition branch'i setup guidance reply ile sonlanir. Bu sayede yanlis veya bos collection ID ile hatali ticket uretimi engellenir.

---

## Basari ve Hata UX

### Basari Reply

Basarili calismada kullaniciya su bilgiler dondurulur:

- Ticket ID
- Category
- Priority
- Assigned Team
- Ticket'in Collections icinde olusturuldugu bilgisi

### Setup Guidance Reply

Eksik config durumunda reply ozellikle su maddeleri soyler:

- `supportCollectionId` eksikse set edilmesi gerekir
- `notificationEmail` eksikse set edilmesi gerekir
- Gmail credential'in iki Gmail node'una baglanmasi gerekir

Bu branch mutasyon yapmadan biter. Yani ne Collections kaydi ne de Gmail bildirimi olusur.

---

## Kullanim Notlari

- Bu akis ticket lifecycle yonetimi yapmaz.
- `close`, `reopen`, `comment`, `status transition` gibi operasyonlar v1 disindadir.
- Musteriye otomatik cevap yazilmaz.
- `Collections` record schema adlari README ile JSON'da birebir tutulmalidir.
- `priority` son karari AI triage sonucu verir; gecersiz veya bos ise formdaki `urgency` fallback olarak kullanilir.
- `urgentNotificationEmail` bos birakilabilir. Bu durumda urgent branch yine calisir ama mail `notificationEmail` adresine gider.

---

## Onerilen Modeller

- `gpt-4o`
- `claude-sonnet-4-6`

Hiz ve maliyet onceligi olan kurulumlarda daha hizli bir model ile triage yapilabilir. Daha muhafazakar siniflandirma gereken enterprise senaryolarda daha guclu bir model tercih edilebilir.

---

## Test Senaryolari

Template import edildikten sonra asagidaki senaryolarla test edilmesi onerilir:

1. `supportCollectionId` bos
   Setup guidance reply donmeli
2. `notificationEmail` bos
   Setup guidance reply donmeli
3. Normal priority support request
   Collections record olusmali ve standard Gmail branch'i calismali
4. High veya critical issue
   Collections record olusmali ve urgent Gmail branch'i calismali
5. LLM structured output
   `triageCategory`, `triagePriority`, `triageSummary`, `triageSuggestedTeam`, `triageNextAction` state'e yazilmali

---

## Lisans

MIT License
