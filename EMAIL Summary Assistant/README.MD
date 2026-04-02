# EMAIL Summary Assistant

Bu template Gmail inbox icindeki son donemi ozetler, LLM ile gun sonu email ozeti uretir ve sonucu Gmail uzerinden gonderir.

## Akis

1. Guncel tarihi alir.
2. Gmail'den son donemdeki inbox mesajlarini ceker.
3. Mesajlari LLM'e uygun JSON metnine donusturur.
4. Aksiyon odakli bir gun sonu ozeti olusturur.
5. Ozeti belirledigin email adresine yollar.

## Duzenlenmesi Gereken Alanlar

Import ettikten sonra su ayarlari yap:

- `Generate Email Summary` node'unda model sec.
- Iki Gmail node'una da `gmailOAuth2` credential bagla.
- `Nightly Email Summary` start node icindeki `startState.summaryRecipient` degerini kendi alici adresinle degistir.

Istersen su varsayilanlari da guncelleyebilirsin:

- `startState.subjectPrefix`
- `startState.gmailQuery`
- `startState.lookbackDays`

Varsayilan Gmail query:

`newer_than:1d in:inbox -category:promotions -category:social -in:sent -in:drafts -in:spam -in:trash`

Bu query son 1 gunun inbox mesajlarini alir ve promotions/social/sent/drafts/spam/trash tarafini disarida birakir.

## Schedule Notu

Bu repo ve `All Nodes` export yapisinda ayri bir schedule node'u gorunmedigi icin schedule bilgisi JSON icine gomulu degil. Bu nedenle flow, scheduler tarafindan tetiklenecek sekilde hazirlandi.

Her aksam calistirmak icin pratik varsayim:

- Saat: `19:00`
- Timezone: `Europe/Istanbul`
- Frekans: her gun

Yani import sonrasi platform scheduler tarafinda bu flow'u her gun saat 19:00'da tetiklemen yeterli.

## Notlar

- `summaryRecipient` olarak ayni Gmail hesabini da verebilirsin; o zaman ozet kendine gelir.
- Ozet maili Turkce ve ASCII karakterlerle uretilir.
- Gmail node export'undaki aciklama metni repo icinde yanlis gorunuyor, ama node girisleri gercekten Gmail aksiyonlari icin tanimli.
