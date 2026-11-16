# Customer Churn Dashboard (Power BI)

## 1. Alətlər və dataset

- **Alət:** Power BI Desktop (Windows).
- **Dataset:** Kaggle, *Telco Customer Churn* (`WA_Fn-UseC_-Telco-Customer-Churn.csv`), tapşırıqda təklif olunan dataset. 7 043 müştəri, 21 sütun.
- **Qayda:** bütün təmizləmə və tip dəyişmələri **Power Query**-də, bütün hesablamalar **DAX**-da edilib. Data Python və ya Excel-də əvvəlcədən hazırlanmayıb.

## 2. Power Query qərarları

### 2.1 TotalCharges

`TotalCharges` mətn kimi import olunur və `tenure = 0` olan 11 yeni müştəridə dəyər boş sətirdir (boşluq simvolu).

**Qərar: bu dəyərlər 0-a çevrildi, sətirlər silinmədi.**

Əsaslandırma:
- Bu müştərilər yenicə qoşulublar və hələ faktura almayıblar, yəni real ümumi ödəniş 0-dır. 0 məntiqi olaraq düzgün dəyərdir.
- Sətirləri silsəydim, `Total Customers` 7 043 yox 7 032 olardı və 11 yeni müştəri hesabatdan itərdi. Üstəlik bu müştərilər churn etməyiblər, silmək churn rate-i bir qədər şişirdərdi.

Texniki həll: addım `Table.TransformColumns` ilə edilib: `Text.Trim` ilə boşluq təmizlənir, boş dəyər 0 olur, qalanları `Number.FromText(_, "en-US")` ilə rəqəmə çevrilir. `"en-US"` seçilib ki, kompüterin regional ayarı (onluq ayırıcı vergül) nöqtəli rəqəmləri (`29.85`) səhv oxumasın.

### 2.2 Data tipləri

Hər sütuna açıq tip verilib (`Set data types` addımı):
- **Text:** customerID, gender, Partner, Dependents, PhoneService, MultipleLines, InternetService, OnlineSecurity, OnlineBackup, DeviceProtection, TechSupport, StreamingTV, StreamingMovies, Contract, PaperlessBilling, PaymentMethod, Churn
- **Whole Number:** SeniorCitizen, tenure
- **Fixed decimal number (Currency):** MonthlyCharges, TotalCharges

### 2.3 Yeni sütunlar

| Sütun | Qayda |
|---|---|
| `ChurnFlag` | Churn = "Yes" olarsa 1, əks halda 0 (Whole Number) |
| `TenureBand` | tenure üzrə: `0-12`, `13-24`, `25-48`, `49+` |
| `ChargeBand` | MonthlyCharges kvartilləri: `Q1 Low`, `Q2 Mid-Low`, `Q3 Mid-High`, `Q4 High` |

`ChargeBand` üçün kvartil sərhədləri sabit rəqəm kimi yazılmayıb, `List.Percentile` ilə datadan hesablanır (`MonthlyList`, `Q1`, `Q2`, `Q3` addımları). Beləliklə data dəyişsə, band sərhədləri avtomatik yenilənir.

### 2.4 Applied Steps sırası

`Source` → `Promoted Headers` → `TotalCharges blank to 0` → `Set data types` → `ChurnFlag` → `TenureBand` → `MonthlyList / Q1 / Q2 / Q3` → `AddChargeBand`.

Sıra məntiqidir: əvvəl təmizləmə və tiplər, sonra tiplənmiş sütunlara əsaslanan yeni sütunlar. `ChargeBand` `MonthlyCharges`-in rəqəm tipinə ehtiyac duyur, ona görə tiplərdən sonra gəlir.

## 3. DAX measure-ləri

Hamısı açıq (explicit) measure-dir, `Telco` cədvəlindədir.

```
Total Customers = COUNTROWS(Telco)

Churned Customers = CALCULATE([Total Customers], Telco[Churn] = "Yes")

Churn Rate % = DIVIDE([Churned Customers], [Total Customers])

Avg Monthly Charges = AVERAGE(Telco[MonthlyCharges])

Monthly Revenue at Risk = CALCULATE(SUM(Telco[MonthlyCharges]), Telco[Churn] = "Yes")
```

**Niyə `DIVIDE`, `/` yox?** `/` operatoru ilə məxrəc 0 olduqda xəta alınır. Slicer və filtrlərdən sonra bəzi seqmentdə heç bir müştəri qalmaya bilər, onda `Total Customers = 0` olur. `DIVIDE` belə halda xəta yox, boş (blank) qaytarır, hesabat sınmır.

**Niyə churn rate measure-dir?** Seqmentləri müqayisə edəndə sətir sayı yanıldır (böyük seqmentdə churn sayı həmişə çox olur). Faiz isə hər seqmentin öz ölçüsünə görə hesablanır. Measure olduğu üçün slicer və klik filtrlərində də düzgün qalır, hər seqment üçün öz filtrinə görə yenidən hesablanır.

Qeyd: heç bir müştərinin churn etmədiyi seqmentlərdə `Churned Customers` boş çıxır və churn rate də boş görünür (0% yox). Bu, cədvəldə (məsələn, `Two year / DSL / Mailed check`) görünür və 0 churn deməkdir.

## 4. Dashboard

Bir səhifəlik interaktiv hesabat:
- **KPI kartları:** Total Customers, Churn Rate %, Monthly Revenue at Risk, Avg Monthly Charges.
- **Qrafiklər (churn rate %, sətir sayı yox):** Contract üzrə, InternetService üzrə, PaymentMethod üzrə (bar chart), TenureBand üzrə (column chart, `0-12 → 49+` sırası ilə, əlifba və ya dəyər sırası ilə yox).
- **Slicer-lər:** Contract, InternetService, TenureBand. Slicer-lər səhifədəki bütün kartları və qrafikləri filtrləyir, bir çubuğa klik də digər vizualları filtrləyir.

Cross-filter yoxlanışı: Contract slicer-də `One year` seçiləndə kartlar və qrafiklər yenilənir: churn rate 26,5%-dən 11,3%-ə düşür, Monthly Revenue at Risk $139K-dan $14K-ya enir (`churn_dashboard_slicer.png`).

## 5. Əsas tapıntılar

Ümumi göstəricilər: **7 043 müştəri, churn rate 26,5%, aylıq riskdə olan gəlir $139 131, orta aylıq ödəniş $64,76.**

| Ölçü | Seqment | Churn rate |
|---|---|---|
| Contract | Month-to-month | 42,7% |
| | One year | 11,3% |
| | Two year | 2,8% |
| InternetService | Fiber optic | 41,9% |
| | DSL | 19,0% |
| | No | 7,4% |
| PaymentMethod | Electronic check | 45,3% |
| | Mailed check | 19,1% |
| | Bank transfer (automatic) | 16,7% |
| | Credit card (automatic) | 15,2% |
| TenureBand | 0-12 | 47,4% |
| | 13-24 | 28,7% |
| | 25-48 | 20,4% |
| | 49+ | 9,5% |

Nəticə: müqavilə növü ən güclü faktordur. Month-to-month müqavilədən Two year-a keçdikdə churn rate 42,7%-dən 2,8%-ə düşür. Fiber optic və Electronic check yüksək churn ilə, ilk 12 ay isə ən böyük churn riski ilə bağlıdır.

## 6. Ən riskli iki seqment

Seqmentləri Contract × InternetService × PaymentMethod kombinasiyaları üzrə cədvəldə (`segment_table`) yoxladım.

**Seçim qaydası:** yalnız ən azı **250 müştərisi** olan seqmentlərə baxdım (müştəri bazasının təxminən 3,5%-i). Çünki kiçik seqmentdə churn rate etibarsızdır: bir neçə müştəri faizi bir neçə bənd dəyişə bilər. Üstəlik belə seqmentin ümumi gəlirə təsiri azdır. Məsələn, 3 müştərili seqmentdə 100% churn əslində bir neçə nəfərin tarixçəsidir, bunun üzərində kampaniya qurmaq və resurs ayırmaq düzgün olmaz.

| # | Seqment | Müştəri | Churn rate | Monthly Revenue at Risk |
|---|---|---|---|---|
| 1 | Month-to-month + Fiber optic + Electronic check | 1 307 (bazanın 18,6%-i) | **60,4%** | **$68 282** (bütün riskin 49,1%-i) |
| 2 | Month-to-month + Fiber optic + Bank transfer (automatic) | 327 (bazanın 4,6%-i) | **45,6%** | **$13 062** (riskin 9,4%-i) |

Digər diqqət çəkən seqmentlər: Month-to-month + Fiber + Mailed check 50,7% (yalnız 201 müştəri, ona görə 250 həddinə düşmür), Month-to-month + Fiber + Credit card 41,6% (293 müştəri, $10 731).

**Ən vacib tapıntı:** Month-to-month + Fiber optic müştəriləri (ödəniş növündən asılı olmayaraq 4 seqment: 2 128 müştəri) bazanın cəmi təxminən 30%-ni təşkil edir, amma bütün aylıq riskin təxminən **72%-ni** ($100 482) verir.

## 7. Tövsiyələr (prioritet sırası ilə)

### 1. Prioritet: Month-to-month Fiber optic müştərilərini, ilk növbədə Electronic check ödəyənləri, uzunmüddətli müqaviləyə keçirmək

- **Hədəf:** Seqment 1 (1 307 müştəri, 60,4%, $68 282 risk).
- **Əsas:** eyni seqmentdə (Fiber optic + Electronic check) müqavilə növünə görə churn rate: Month-to-month 60,4%, One year 26,0%, Two year 9,8% (yalnız 92 müştəri, ona görə ehtiyatla oxunmalıdır).
- **Hərəkət:** bu müştərilərə birbaşa əlaqə: 1 illik və ya 2 illik müqavilə üçün endirim və ya pulsuz ay təklifi, eyni zamanda Fiber xidmətinin keyfiyyət problemlərini yoxlamaq (əlaqə zamanı şikayət səbəblərini toplamaq).
- **Təxmini effekt:** bu seqmentdə churn 10 faiz bəndi azalsa (təxminən 131 müştəri), aylıq təxminən $11 000 gəlir qorunar (hesablama: seqmentdə churn edən müştərinin orta aylıq ödənişi təxminən $86,5). Bu, rəqəmlərdən çıxan təxmini hesabdır, proqnoz deyil.
- **Niyə birinci:** həm ən yüksək rate, həm də ən böyük həcm və gəlir riski bir seqmentdədir. Bir kampaniya ilə riskin yarısına çatmaq olar.

### 2. Prioritet: Electronic check-dən avtomatik ödənişə keçid

- **Əsas:** Electronic check bütün qruplarda ən yüksək churn göstərir. Month-to-month Fiber qrupunda Electronic check 60,4%, Bank transfer 45,6%, Credit card 41,6%; Month-to-month DSL qrupunda Electronic check 40,5%, Bank transfer 19,8%, Credit card 27,0%. Ümumi göstərici: Electronic check 45,3%, avtomatik ödəniş üsulları 15-17%.
- **Hərəkət:** Electronic check istifadə edən müştərilərə avtomatik ödənişə keçid üçün kiçik təşviq (məsələn, bir aylıq kiçik endirim). Seqment ölçüsü: Month-to-month DSL + Electronic check (474 müştəri, 40,5%, $8 776) və yuxarıdakı Fiber seqmenti.
- **Qeyd:** bu korrelyasiyadır. Ödəniş üsulunu dəyişmək özü churn-ü azaltmaya bilər, ola bilər ki, ödənişdən narazı müştərilər elektron çek ilə ödəyirlər. Ona görə bu hərəkət kiçik pilot kimi sınanmalıdır.

### 3. Prioritet: İlk 12 ay üçün onboarding proqramı

- **Əsas:** `0-12` ay müştərilərinin churn rate-i 47,4%, `49+` ay müştərilərində isə 9,5%-dir. İlk il ən kritikdir.
- **Hərəkət:** yeni müştərilərə 30/60/90-cu günlərdə əlaqə (xidmətdən razılıq, texniki problem yoxlanışı), xüsusilə yeni Month-to-month müştəriləri üçün. Müqaviləni ilk üç ayda uzunmüddətliyə keçirmək təklifi.
- **Seqment ölçüsü:** `0-12` ay seqmenti təxminən **2 000** müştəridən ibarətdir (kartda `2K`), churn rate 47,4%, aylıq risk təxminən **$69 000** (kartda `$69K`). Bu, bütün aylıq riskin ($139 131) təxminən yarısıdır, yəni ilk 12 ayda gedən müştərilər gəlir itkisinin təxminən yarısını verir. Bu seqmentdə orta aylıq ödəniş $56,10-dur. Rəqəmlər kartlardakı yığcam göstərilişə görə yuvarlaqlaşdırılıb.
- **Əlavə tapıntı:** `0-12` slicer-i seçili ikən digər qrafiklər: Month-to-month 51,4% (One year 10,5%), Fiber optic 69,9% (DSL 40,0%, No 17,9%), Electronic check 62,0% (Bank transfer 45,0%, Credit card 36,0%, Mailed check 32,8%). Yəni yeni Fiber optic müştəriləri və Electronic check ilə ödəyən yeni müştərilər ən riskli qrupdur, bu da 1-ci və 2-ci tövsiyələrlə üst-üstə düşür.

### Aşağı prioritet

Two year müqaviləli (2,8%) və internet xidməti olmayan (7,4%) müştərilərin churn riski aşağıdır. Bu qruplarda xüsusi kampaniya qurmağa ehtiyac yoxdur, resursları yuxarıdakı seqmentlərə yönəltmək daha səmərəlidir.

## 8. Məhdudiyyətlər

- Dataset müştərinin **niyə** getdiyini göstərmir (qiymət, xidmət keyfiyyəti, rəqib təklifi). Tapıntılar korrelyasiyadır, səbəb-nəticə deyil.
- Üç ölçülü seqmentlərdə (Contract × InternetService × PaymentMethod) bəzi qruplar kiçikdir, ona görə yalnız ≥250 müştərisi olan seqmentlərə əsaslandım.
- `ChargeBand` sütunu yaradılıb, amma bu hesabatda vizual kimi istifadə olunmayıb.
- Churn rate və revenue at risk eyni dövr datasından (snapshot) hesablanıb, zaman üzrə tendensiya yoxdur.
- Kartlarda böyük rəqəmlər (`7K`, `$139K`) yığcam göstərilir. Dəqiq dəyərlər: 7 043 müştəri, $139 131.

## 9. Təhvil fayllar

| Fayl | Məzmun |
|---|---|
| `churn_dashboard.pbix` | Power BI hesabatı |
| `churn_dashboard_page.png` | Hesabat səhifəsi, slicer seçilməmiş |
| `churn_dashboard_slicer.png` | Eyni səhifə, Contract = One year seçilmiş |
| `note.md` | Bu fayl |
