# 1507 - TUBITAK KOBI Ar-Ge Baslangic Destek Programi Arge Dokuman Olusturucu

### Creator : [selim@agentron.ai](selim@agentron.ai)

Bu agentflow, **TUBITAK 1507 KOBI Ar-Ge Baslangic Destek Programi** icin ilk taslak Ar-Ge dokumanini olusturmak uzere tasarlanmis bir ornektir. Akis, kullanicidan proje bilgilerini form uzerinden toplar, girilen verilerin dokuman uretimi icin yeterli olup olmadigini kontrol eder ve yeterli bilgi varsa 1507 basvuru mantigina uygun bir taslak dokuman uretir.

---

## Overview

Bu akis iki temel problemi cozer:

- Basvuru oncesi daginik proje bilgisini tek bir standart formatta toplar
- Bilgi eksikliginde kullaniciyi dogrudan taslak uretmek yerine eksik alanlara yonlendirir

Boylece ekip, zayif veya belirsiz girdilerle dusuk kaliteli bir basvuru taslagi almak yerine once eksiklerini gorur; yeterli girdide ise yapilandirilmis bir Ar-Ge dokumani elde eder.

---

## Flow Mimarisi

Asagidaki akis, surecin mantigini ozetler:

```text
Baslangic Formu
   |
   v
Check 1507 Readiness
   |
   +--> Bilgi eksik / belirsiz
   |        |
   |        v
   |   Generate Missing Information Guidance
   |
   +--> Bilgi yeterli / taslaga hazir
            |
            v
      Build Proposal Strategy
            |
            v
      Generate 1507 Draft Document
```

### Adimlar

1. **Baslangic Formu** node'u proje ile ilgili temel basvuru verilerini toplar.
2. **Check 1507 Readiness** node'u, girilen bilgilerin dokuman uretimi icin yeterli olup olmadigini degerlendirir.
3. Bilgi yetersizse **Generate Missing Information Guidance** node'u eksik alanlari, neden onemli olduklarini ve kullanicidan hangi sorularin istenmesi gerektigini listeler.
4. Bilgi yeterliyse **Build Proposal Strategy** node'u proje kurgusunu netlestirir; problem, yenilik, teknik yaklasim, ciktilar ve ticarilesme cercevesini hazirlar.
5. **Generate 1507 Draft Document** node'u, onceki analiz ve kullanici girdileriyle 1507 formatina yakin bir taslak dokuman uretir.

---

## Node Aciklamalari

| Node | Tip | Aciklama |
|------|-----|----------|
| **Start Intake Form** | Start | Form input ile proje girdilerini toplar. |
| **Check 1507 Readiness** | ConditionAgent | Bilginin yeterli ve acik olup olmadigini kontrol eder. |
| **Generate Missing Information Guidance** | LLM | Eksik bilgi listesi, netlestirme sorulari ve iyilestirme onerileri uretir. |
| **Build Proposal Strategy** | LLM | Basvurunun cekirdek kurgusunu ve anlatim omurgasini cikarir. |
| **Generate 1507 Draft Document** | LLM | 1507 icin kullanilabilecek taslak Ar-Ge dokumanini markdown formatinda uretir. |

---

## Formda Toplanan Bilgiler

Akis, asagidaki temel alanlari toplar:

- Firma adi
- Proje adi
- Sektor
- Musteri problemi
- Cozum ve yenilik
- Ar-Ge amaci
- Teknik yaklasim
- Is paketleri
- Beklenen ciktilar
- Hedef pazar
- Ticari etki
- Ekip yetkinligi
- Proje suresi
- Butce ozeti
- Riskler
- Basari kriterleri

Bu alanlar, hem yeterlilik kontrolunde hem de nihai dokuman uretiminde kullanilir.

---

## Uretilen Dokuman Yapisi

Son node, asagidaki basliklari iceren bir taslak uretir:

- Proje Ozeti
- Problem Tanimi ve Ihtiyac
- Yenilikci Yon ve Ar-Ge Niteligi
- Teknik Yaklasim ve Yontem
- Is Paketleri ve Zaman Plani
- Beklenen Ciktilar ve Basari Kriterleri
- Ticarilesme Potansiyeli
- Proje Ekibi ve Yetkinlikler
- Riskler ve Onlemler
- Butce Gerekcesi
- Varsayimlar ve Netlestirilmesi Gereken Noktalar

Bu yapi, resmi basvuru formunun birebir kopyasi degil; 1507 basvurusunu hazirlamayi kolaylastiran bir ilk taslak iskeleti sunar.

---

## Kurulum

### Gereksinimler

- [EmploidAI](https://app.emploid.ai)
- En az bir LLM provider baglantisi

### Import Adimlari

1. EmploidAI'a giris yapin.
2. **Agentflows** alanina gidin.
3. Import islemini baslatin.
4. `1507 - TUBITAK KOBI Ar-Ge Baslangic Destek Programi Arge Dokuman Olusturucu Agent Flow.json` dosyasini import edin.
5. LLM node'larinda kullanmak istediginiz modeli secin.
6. Flow'u calistirip form alanlarini doldurun.

---

## Yapilandirma

### LLM Modeli

Tum LLM node'larinda ayni model kullanilabilecegi gibi, asagidaki gibi bir ayrim da yapilabilir:

- Hazirlik ve kontrol adimlari icin hizli bir model
- Nihai dokuman uretimi icin daha guclu bir model

Onerilen modeller:

- `gpt-4o`
- `claude-sonnet-4-6`

### Prompt Ozellestirme

Kuruma veya sektore gore asagidaki alanlar kolayca ozellestirilebilir:

- 1507 yerine farkli destek programlari
- Sektor-ozel degerlendirme kriterleri
- Dokuman basliklari
- Cikti dili ve ton

---

## Kullanim Senaryolari

- Danismanlik ekiplerinin basvuru oncesi hizli ilk taslak uretmesi
- KOBI'lerin proje fikirlerini 1507 mantigina gore yapilandirmasi
- Is gelistirme veya Ar-Ge ekiplerinin eksik bilgi analizini standartlastirmasi
- Uzun basvuru metinlerinden once "hazirlik seviyesi kontrolu" yapilmasi

---

## Notlar

- Bu akis, resmi basvuru sonucunu tahmin etmez; yalnizca taslak olusturma surecini hizlandirir.
- Bilgi yetersizse dokuman uretmek yerine eksik alanlari gostermesi, bilerek tasarlanmis bir davranistir.
- Istenirse bu akis sonraki versiyonda `CreatePDF`, `Drive` veya `HumanInput` node'lariyla genisletilebilir.

---

## Lisans

MIT License
