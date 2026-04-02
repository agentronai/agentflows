# 1507 - TUBITAK KOBI Ar-Ge Baslangic Destek Programi Arge Dokuman Olusturucu

### Creator : [selim@agentron.ai](selim@agentron.ai)

Bu agentflow, **TUBITAK 1507 KOBI Ar-Ge Baslangic Destek Programi** icin AGY100/AGY101 dokuman basliklarina hizali, cok adimli bir taslak olusturma akisidir. Akis; form verisini toplar, bilgi yeterliligini kontrol eder, her ana bolumu ayri bir LLM ile yazar, sonucu tek markdown dokumanda birlestirir, PDF uretir ve Workspace Drive'a kaydeder.

---

## Overview

Bu tasarim tek bir uzun prompt yerine bolum-bazli uzmanlasmis yazim kullanir:

- Her ana bolum farkli bir LLM node'u tarafindan uretilir
- Butce bolumu `F.1`-`F.7` icin ayri ayri ele alinir
- Son cikti tek markdown dokumanda birlestirilir
- Dokuman PDF'e cevrilir ve Drive'da proje klasorune kaydedilir
- Bilgi eksikligi varsa akis PDF uretmeden once kullaniciyi geri yonlendirir

---

## Flow Mimarisi

```text
Start Intake Form
   |
   v
Check 1507 Readiness
   |
   +--> Eksik bilgi
   |       |
   |       v
   |   Generate Missing Information Guidance
   |
   +--> Yeterli bilgi
           |
           v
      LLM-A  -> A bolumu
      LLM-B  -> B bolumu
      LLM-C  -> C bolumu
      LLM-D  -> D bolumu
      LLM-E  -> E bolumu
      LLM-F1 -> F.1 bolumu
      LLM-F2 -> F.2 bolumu
      LLM-F3 -> F.3 bolumu
      LLM-F4 -> F.4 bolumu
      LLM-F5 -> F.5 bolumu
      LLM-F6 -> F.6 bolumu
      LLM-F7 -> F.7 bolumu
      LLM-F-Merge -> F ana bolumu
      LLM-G  -> Ekler checklist
           |
           v
      LLM-Final-Merge
           |
           v
       Create PDF
           |
           v
      Capture PDF Base64
           |
           v
      Create Root Folder
           |
           v
      Create Project Folder
           |
           v
      Save PDF To Drive
           |
           v
        Success Reply
```

---

## Node Gruplari

| Grup | Node Tipi | Amac |
|------|-----------|------|
| **Start Intake Form** | Start | AGY100/AGY101 basliklarina hizali proje girdilerini form ile toplar. |
| **Check 1507 Readiness** | ConditionAgent | Kritik alanlarin doluluk ve anlam yeterliligini kontrol eder. |
| **Generate Missing Information Guidance** | LLM | Eksik alanlari, sorulmasi gereken sorulari ve iyilestirme onerilerini uretir. |
| **LLM-A ... LLM-G** | LLM | Her ana bolumu veya alt butce bolumunu ayri uzman node ile yazar. |
| **LLM-F-Merge** | LLM | `F.1`-`F.7` ciktilarini tek `F` bolumunde toplar. |
| **LLM-Final-Merge** | LLM | `A`-`G` bolumlerini dogru sirada tek markdown belgeye donusturur. |
| **Create PDF** | CreatePDF | Nihai markdown dokumani PDF'e cevirir. |
| **Capture PDF Base64** | CustomFunction | PDF node ciktisini Drive yazimi icin state'e alir. |
| **Create Root Folder / Create Project Folder / Save PDF To Drive** | Drive | PDF'i `/Workspace/TUBITAK 1507/<proje>` yoluna kaydeder. |
| **Success Reply** | DirectReply | Kaydedilen dosya yolunu kullaniciya bildirir. |

---

## Formda Toplanan Ana Alanlar

Akis; dokuman basliklarina hizali olarak su alanlari toplar:

- Kimlik ve genel bilgiler: `projeNumarasi`, `projeAdi`, `kurulusAdi`, `ortakAdi`
- Teknoloji ve konum: `teknolojiGrubu`, `baslangicThs`, `hedefThs`, `argeSektoru`, `ciktiSektoru`, `hedefPazar`
- `A.0` karsilastirma notlari
- `A`, `B`, `C`, `D`, `E` ana bolumleri icin serbest metin girdileri
- `F.1`-`F.7` butce alt formlari icin ayri serbest metin girdileri
- `G` icin ekler notlari

Bu tasarim tablo bazli maliyet girisi yerine, her alt formu anlatimsel veriyle besleyen pratik bir yapi kullanir.

---

## Flow State

Start node'unda asagidaki state key'leri tanimlidir:

- `sectionA`, `sectionB`, `sectionC`, `sectionD`, `sectionE`
- `sectionF1` ... `sectionF7`
- `sectionF`, `sectionG`
- `finalMarkdown`
- `projeKlasoruAdi`, `pdfDosyaAdi`, `pdfBase64`

Her LLM node'u kendi bolum ciktisini `{{ output }}` ile ilgili state key'ine yazar. Final merge node'u ise bu state key'lerini birlestirerek nihai markdown belgeyi olusturur.

---

## PDF ve Drive Davranisi

Hazirlik kontrolu olumluysa akis:

1. Bolumleri markdown olarak olusturur
2. Tum bolumleri tek belgeye birlestirir
3. `CreatePDF` ile PDF uretir
4. `/Workspace/TUBITAK 1507` ust klasoru altinda proje klasoru olusturur
5. PDF'i su kalipta kaydeder:

`/Workspace/TUBITAK 1507/<projeNumarasi veya projeAdi>/<projeNumarasi veya projeAdi - TUBITAK 1507 Taslak.pdf>`

---

## Kurulum

### Gereksinimler

- [EmploidAI](https://app.emploid.ai)
- En az bir LLM provider baglantisi
- Workspace Drive yetkisi

### Import Adimlari

1. EmploidAI'a giris yapin.
2. **Agentflows** ekranina gidin.
3. Import islemini baslatin.
4. `1507 - TUBITAK KOBI Ar-Ge Baslangic Destek Programi Arge Dokuman Olusturucu Agent Flow.json` dosyasini import edin.
5. Tum `LLM` ve `ConditionAgent` node'larinda model secimini yapin.
6. Workflow'u calistirip formu doldurun.

---

## Kullanim Notlari

- Bu akis resmi basvuru yerine gecen nihai belge uretmez; ilk taslak ve ic calisma metni uretir.
- Bilgi eksikse sistem bilerek PDF olusturmaz.
- `G. Ekler` bolumu dosya yukleme yapmaz; ekler checklist'i olusturur.
- `F.1`-`F.7` butce bolumleri ayri node'larda yazildigi icin ilgili prompt'lar kolayca ozellestirilebilir.

---

## Onerilen Modeller

- `gpt-4o`
- `claude-sonnet-4-6`

Hiz oncelikli kurulumlarda butce alt bolumleri icin daha hizli model, final merge icin daha guclu model tercih edilebilir.

---

## Lisans

MIT License
