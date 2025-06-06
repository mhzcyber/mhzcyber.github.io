---
title: "Fuzzing Basics P1 Arabic"
classes: wide
header:
  teaser: /assets/images/vredar/coverage-fuzzing-visualization.gif
ribbon: red
description: "الجزء الأول من أساسيات الفازينغ و الجزء الأول نظري"
tags: vred-ar
toc:  true
---

<style>
.rtl-content {
  direction: rtl;
  text-align: right;
  unicode-bidi: bidi-override;
}
</style>

# أساسيات الفازينغ

<div class="rtl-content" markdown="1">

السلام عليكم
شرح ال fuzzing و رح يكون أول جزء نظري
و ثاني جزء عملي رح نعمل fuzz على LOLCODE compiler كنت جبت فيه اكتر heap overflow و بلغتهم من شهر 11 السنة الماضية بس مافي رد و ما اعتقد رح يكون في رد
هو fuzzing هيك شغلات فقط للمرح و التسلية.

أساسيات و تقنيات ال fuzzing

ال fuzzing أغلب الناس بتعرفه خاصة بال pentest لو شغال على ويب 
نفس المبدا الاساسي بس بكمية تفاصيل اكثر و متقدم اكثر بكثير.

ال fuzzing ما بس جزء من ال cybersecurity كمان بستخدم بال Software QA 
الفكرة انك عم تجرب تعطي مدخلات مختلفة, خاطئة و صحيحة و داتا عشوائية للبرنامج

كمثال:

</div>

```bash
ls temp_dir 
```

<div class="rtl-content" markdown="1">

البرنامج ls و المدخل هو temp_dir 
منجرب اشكال مختلفة من المدخلات و منجرب مع flags مختلفة.

الهدف طبعا انك توصل لثغرات, crashes, تسريبات بالذاكرة memory leaks, او اي مشاكل تانية ممكن تطلع بالبرنامج.

# الأجزاء الأساسية لل fuzzing:

## 1- Seeds/Corpus 
ببساطة هي المدخلات, وهي بتكون المدخلات الأولية الي انت بتجمعها او بتعملها بطرق مختلفة و هي تسمى seeds 
- مقال و فيديو لاحق عن الموضوع هاد.
اما ال corpus الي اداة ال fuzzing - fuzzer بأنشئها اثناء الفازينغ
الي منركز عليه انو ال seeds او المدخلات:
- تخلي اجزاء مختلفة من البرنامج تشتغل
- تكون المدخلات minimal بس كافية, بمعنى كل ماقدرت تصغر حجم المدخل seed الي بخلي الثغرة او الكراش يصير, بكون أحسن.
- المدخلات تخلي behaviors مختلفة بالبرنامج تتنفذ

## 2- قياس التغطية
ال coverage measurement, التغطية او ال coverage هو المؤشر لكم سطر برمجي, جزء من البرنامج قدر الfuzzer يصله و ينفذه
كل ما ال fuzzer قدر يغطي مساحة اكبر من البرنامج كل ما بعني انه رح يجرب مدخلات مختلفة باماكن مختلفة من البرنامج و بالتالي بعني احتمالية انه ي trigger كراش او ثغرات اكتر.

هون منركز على :

- line coverage كم سطر برمجي وصلناله
- branch coverage كم برانش متل if/else وصلناله
- path coverage كم execution paths وصلناله
- block coverage كم جزء برمجي او block تم تنفيذه و وصلناله

طبعا التغطية جدا مهمة لل smart fuzzers و هي الشيء الاساسي الي بخلي الfuzzing ذكي

![](/assets/images/vredar/coverage-fuzzing-visualization.gif)

## 3- minimization/trimming 
انت بس رح تحاول تقلل المدخلات, تصغرها, ما تكون متكررة, طبعا مع اخذ بالاعتبار انو لسا المدخلات ت trigger behaviors

العملية هي المفروض:

- تمسح اي تكرارات او مدخلات مالها علاقة او لازمة
- تساعد بالتنقيح debugging بانو المدخل ينفذ trigger بس ال paths الي الها علاقة بالكراش او الثغرة
- تحسن كفاءة ال fuzzing بانك تركز على المدخلات او اجزاء من المدخلات الي بتجيب اعلى تغطية

عادة ال fuzzers المتطورة بتعمل هي العملية بنفسها.

## 4- تقنيات التلاعب Mutation
"ارحم الترجمة 😂" الفكرة انو منستخدم تقنيات معينة لنتلاعب بالمدخلات لحتى نعمل مدخلات جديدة

- Bit flips تغير بيت واحد بالمدخل
- Arithmetic operations تعمل عمليات حسابية بانك تنقص او تزيد على المدخل seed
- Block operations اضيف, تحذف, او تكرر جزء من المدخل seed
- Dictionary-based mutations بتستخدم قاموس من المدخلات او القيم المعروفة و بصير ال fuzzer بتلاعب فيهاو بقلبها
- structured mutations انو يكون عندك كمثال شي اسمه grammer-aware بكون في متل تيمبلت في كمية من القواعد الي على اساسهابتم التلاعب و تغير المدخلات

## 5- Instrumentation
عبارة عن كود بنضاف للبرنامج الي رح تعمل عليه فازينغ, شغلة الكود انه يجمع معلومات و يتعقب التنفيذ execution الخاصة بالبرنامج

ال instrumentation بتعطينا شوية ميزات:

- بتتعقب معلومات التغطية
- بتساعد بانو نلقط اي مشاكل بالميموري
- بتراقب وقت التنفيذ
- بتساعد بتحديد الكراشز و ال hangs 

هذا والله اعلم 💚

الجزء الثاني من السلسلة هي هو التطبيق العملي ان شاء الله


</div>
