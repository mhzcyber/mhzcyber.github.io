---
title: "Fuzzing a LOLCODE Compiler with AFL++ Arabic"
classes: wide
header:
  teaser: /assets/images/vredar/cli-fuzzing.gif
ribbon: red
description: "تطبيق عملي على ايجاد ثغرات في CLI LOLCODE عبر الفازينغ باداة AFL++"
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

# Fuzzing a LOLCODE Compiler with AFL++

<div class="rtl-content" markdown="1">

بأول جزء من الفازينغ, شرحنا الفازينغ و تفاصيل بشكل نظري.
بهي المقال رح نطبق بشكل عملي على الفازينغ و نعمل fuzz على ال compiler للغة برمجة اسمها LOLCODE

لما نجي نحكي عن الفازينغ في كتير أدوات تستخدمها و بتقدر تبرمج  fuzzer بنفسك:

</div>

## أشهر أدوات الفازينغ

<div class="rtl-content" markdown="1">

1. AFL++ (American Fuzzy Lop++) - الأداة الي رح نستخدمها 
2. libFuzzer - مكتبة لل فازينغ و بتجي مضمنة مع LLVM
3. Honggfuzz
4. OSS-Fuzz - أداة من جوجل خاصة للفازينغ على مشارع الأوبن سورس
5. Radamsa - الاداة هي ما منستخدمها لأجل الفازينغ مباشر و انما لاجل عمل المدخلات 
6. Peach Fuzzer

في ادوات فازينغ تانية لأسباب مختلفة و حسب اللغة و حسب الهدف الي رح تعمل عليه فازينغ و حسب النظام الي رح تستخدمه
و اكيد في فروقات بينهم سواء بالسرعة و الأداء و ان شاء الله اشرح التفاصيل هي ب منشور او مقال قادم.

</div>

## تثبيت AFL++

<div class="rtl-content" markdown="1">

عملية الثبيت جدا بسيطة و مشروحة هون: https://github.com/AFLplusplus/AFLplusplus/blob/stable/docs/INSTALL.md

</div>

```bash
sudo apt-get update
sudo apt-get install -y build-essential python3-dev automake cmake git flex bison libglib2.0-dev libpixman-1-dev python3-setuptools cargo libgtk-3-dev
# try to install llvm 14 and install the distro default if that fails
sudo apt-get install -y lld-14 llvm-14 llvm-14-dev clang-14 || sudo apt-get install -y lld llvm llvm-dev clang
sudo apt-get install -y gcc-$(gcc --version|head -n1|sed 's/\..*//'|sed 's/.* //')-plugin-dev libstdc++-$(gcc --version|head -n1|sed 's/\..*//'|sed 's/.* //')-dev
sudo apt-get install -y ninja-build # for QEMU mode
sudo apt-get install -y cpio libcapstone-dev # for Nyx mode
sudo apt-get install -y wget curl # for Frida mode
sudo apt-get install -y python3-pip # for Unicorn mode
git clone https://github.com/AFLplusplus/AFLplusplus
cd AFLplusplus
make distrib
sudo make install
```

<div class="rtl-content" markdown="1">

لما يخلص التثبيت لازم تلاقي كل اوامر afl موجودة:

</div>

```bash
 afl-
afl-addseeds           afl-clang-fast         afl-fuzz               afl-ld-lto             afl-plot
afl-analyze            afl-clang-fast++       afl-g++                afl-lto                afl-showmap
afl-c++                afl-clang-lto          afl-gcc                afl-lto++              afl-system-config
afl-cc                 afl-clang-lto++        afl-gcc-fast           afl-network-client     afl-tmin
afl-clang              afl-cmin               afl-g++-fast           afl-network-server     afl-whatsup
afl-clang++            afl-cmin.bash          afl-gotcpu             afl-persistent-config
```


## Compiling LOLCODE with AFL++ and ASAN

<div class="rtl-content" markdown="1">

بالاول شو يعني تعمل compile مع AFL++?
هون كيف بتصير ال insturmentation الي ذكرناها من قبل

الاداة AFL عندها compiler خاص فيها, بالعادة لما تعمل compile لبرنامج بتستخدم gcc
ال AFL عندها afl-gcc خاص فيها
رح يعمل ال compilation متل ال gcc بس يضيف اجزاء كود اخرى تعمل تعقب او track للتغطية الكود او ال code coverage و ما الى ذلك مما ذكرنا بالمقال السابق.

ال ASAN هو اختصار ل Address Sanatizer
عبارة عن اداة مندمجها بالبرنامج شغلتها تعمل detect لأخطاء الذاكرة متل 

- Buffer overflow
- Use After Free
- Memory leaks
- Double-free
...الخ

بالاول منزل LOLCODE

</div>

```bash
# Clone the lci repository (LOLCODE compiler)
git clone https://github.com/justinmeza/lci.git
cd lci

# Clean any previous builds
make clean
rm -f CMakeCache.txt
```

<div class="rtl-content" markdown="1">

بعدين منعين متغيرات البيئة - environment variables بحيث المجمع compiler يستخدم AFL++ مع ASAN.

</div>

```bash
# Set environment variables for the compiler to use AFL++ with ASAN
export CC=afl-cc
export CXX=afl-c++
export CFLAGS="-fsanitize=address -g -O1 -fno-omit-frame-pointer"
export CXXFLAGS="-fsanitize=address -g -O1 -fno-omit-frame-pointer"
export LDFLAGS="-fsanitize=address"
```

<div class="rtl-content" markdown="1">

- `export CC=afl-cc` and `export CXX=afl-c++`: كرمال نحدد المجمع compilter كرمال يستخدم ال AFL++ Compilter ليبني البرنامج

- `export CFLAGS="-fsanitize=address -g -O1 -fno-omit-frame-pointer"`: هيك عم نضيف خيارات و مميزات للمجمع متل انه يفعل ال ASAN و يضيف معلومات التنقيح - debugging.
  - `-fsanitize=address`: بفعل ال ASAN
  - `-g`: ليضيف معلومات التنقيح debugging information للبرنامج
  - `-O1`: شيء اله علاقة بال optimization ممكن نشرح الموضوع هاد بغير مقال
  - `-fno-omit-frame-pointer`: بتساعد بتتبع الستاك 
- `cmake .`: البناء الاولي للبرنامج باستخدام CMake.
- `make`: لبناء البرنامج 

الان منعمل الكونفيغ و بنعمل build

</div>

```bash
# Configure using CMake
cmake .

# Build the program
make
```

<div class="rtl-content" markdown="1">

أحيانا بصير مشاكل و بكتمل بناء البرنامج بس مابتم دمج ASAN معه ف لازم نتأكد:

</div>

```bash
# Verify ASAN is included in the compiled program
ldd ./lci | grep asan
```


# Let the fuzzing starts

<div class="rtl-content" markdown="1">

بالاول بس منجهز المجلدات لاجل الفازينغ

</div>

```bash
# Create directory structure for fuzzing
mkdir -p ../fuzzing/in ../fuzzing/out
```

## Creating Seeds for fuzzing

<div class="rtl-content" markdown="1">

نحن قلنا من قبل انو ال seeds ببساطة هي المدخلات, وهي بتكون المدخلات الأولية الي انت بتجمعها او بتعملها بطرق مختلفة و هي تسمى seeds

طبعا المدخلات هي لازم تكون صحيحة ما بتعمل أخطاء

قمت بانشاء 4 برامج بسيطة بلغة lol لنعطيهم لل CLI كمدخلات
أيضا كل برنامج من المكتوبين فيه مميزات مختلفة عن التانية, منه بس بطبع جملة بسيطة
التاني فيه متغيرات اكتر, الي بعدها فيه conditionals
الي بصير ان كل مرة ال CLI compiler الخاصة ل LOLCODE بشغل واحد من البرامج هي رح يوصل لأجزء مختلفة من ال CLI
بالتالي منحقق التغطية او ال coverage.

</div>

```bash
# Create a simple "Hello World" program
echo 'HAI
VISIBLE "Hello World"
KTHXBYE' > ../fuzzing/in/hello.lol

# Create a more complex example with variables
echo 'HAI
I HAS A VAR ITZ 10
VISIBLE VAR
KTHXBYE' > ../fuzzing/in/variables.lol

# Create an example with basic conditionals
echo 'HAI 
I HAS A VAR
VISIBLE "Enter a number: "
GIMMEH VAR
BOTH SAEM VAR AN 42, O RLY?
  YA RLY, VISIBLE "The answer!"
  NO WAI, VISIBLE "Not the answer!"
OIC
KTHXBYE' > ../fuzzing/in/conditional.lol

# Create an example with loops
echo 'HAI
I HAS A COUNTER ITZ 0
IM IN YR LOOP UPPIN YR COUNTER TIL BOTH SAEM COUNTER AN 5
  VISIBLE COUNTER
IM OUTTA YR LOOP
KTHXBYE' > ../fuzzing/in/loop.lol
```

## Fuzzer Go

<div class="rtl-content" markdown="1">

منعين متغير لاجل ال ASAN

بعدها منقدر نشغل ال fuzzer

</div>

```bash
# Set ASAN environment variables
export ASAN_OPTIONS=detect_leaks=1:symbolize=1:abort_on_error=1

# Start the fuzzing process
cd ~/lci-fuzz/lci
afl-fuzz -i ../fuzzing/in -o ../fuzzing/out -m none -- ./lci @@
```

<div class="rtl-content" markdown="1">

- `-i ../fuzzing/in`: هاد المجلد في فيه المدخلات الأولية
- `-o ../fuzzing/out`: المجلد الي بتتخزن فيه النتائج
- `-m none`: ب AFL بكون في حد لقديش البرنامج المستخدم مسمحله يخصص و يتسخدم من الذاكرة - الميموري. لما نستخدم ASAN من خبرتي لازم يكون الحد none لانو اصلا ASAN شغلته يلقط المشاكل بالذاكرة.
- `-- ./lci @@`: هاد الامر الي رح ينفذه الfuzzer و رح يبدل ال @@ مع المدخلات.


لما تشغل ال fuzzer احتمال يجيك رسالة مشابهة:

</div>

```bash
afl-fuzz++4.32c based on afl by Michal Zalewski and a large online community
[+] AFL++ is maintained by Marc "van Hauser" Heuse, Dominik Maier, Andrea Fioraldi and Heiko "hexcoder" Eißfeldt
[+] AFL++ is open source, get it at https://github.com/AFLplusplus/AFLplusplus
[+] NOTE: AFL++ >= v3 has changed defaults and behaviours - see README.md
[+] No -M/-S set, autoconfiguring for "-S default"
[*] Getting to work...
[+] Using exploration-based constant power schedule (EXPLORE)
[+] Enabled testcache with 50 MB
[+] Generating fuzz data with a length of min=1 max=1048576
[*] Checking core_pattern...

[-] Your system is configured to send core dump notifications to an
    external utility. This will cause issues: there will be an extended delay
    between stumbling upon a crash and having this information relayed to the
    fuzzer via the standard waitpid() API.
    If you're just experimenting, set 'AFL_I_DONT_CARE_ABOUT_MISSING_CRASHES=1'.

    To avoid having crashes misinterpreted as timeouts, please
    temporarily modify /proc/sys/kernel/core_pattern, like so:

    echo core | sudo tee /proc/sys/kernel/core_pattern

[-] PROGRAM ABORT : Pipe at the beginning of 'core_pattern'
         Location : check_crash_handling(), src/afl-fuzz-init.c:2547
```

<div class="rtl-content" markdown="1">

كل ماعليك او تنفذ الأمر التالي:

</div>

```bash
echo core | sudo tee /proc/sys/kernel/core_pattern
```

<div class="rtl-content" markdown="1">

و ترجع تشغل ال fuzzer مرة تانية و لازم هلق يجيك شي مشابه:

</div>

```bash
            american fuzzy lop ++4.32c {default} (./lci) [explore]
┌─ process timing ────────────────────────────────────┬─ overall results ────┐
│        run time : 0 days, 1 hrs, 58 min, 3 sec      │  cycles done : 8     │
│   last new find : 0 days, 0 hrs, 0 min, 20 sec      │ corpus count : 810   │
│last saved crash : 0 days, 0 hrs, 43 min, 56 sec     │saved crashes : 6     │
│ last saved hang : 0 days, 1 hrs, 18 min, 42 sec     │  saved hangs : 4     │
├─ cycle progress ─────────────────────┬─ map coverage┴──────────────────────┤
│  now processing : 468.53 (57.8%)     │    map density : 0.65% / 1.66%      │
│  runs timed out : 0 (0.00%)          │ count coverage : 5.35 bits/tuple    │
├─ stage progress ─────────────────────┼─ findings in depth ─────────────────┤
│  now trying : havoc                  │ favored items : 88 (10.86%)         │
│ stage execs : 118/450 (26.22%)       │  new edges on : 161 (19.88%)        │
│ total execs : 2.35M                  │ total crashes : 30 (6 saved)        │
│  exec speed : 387.2/sec              │  total tmouts : 881 (0 saved)       │
├─ fuzzing strategy yields ────────────┴─────────────┬─ item geometry ───────┤
│   bit flips : 7/26.8k, 5/26.8k, 6/26.8k            │    levels : 22        │
│  byte flips : 0/3352, 0/3344, 0/3328               │   pending : 12        │
│ arithmetics : 11/234k, 0/465k, 0/462k              │  pend fav : 0         │
│  known ints : 4/30.1k, 1/126k, 0/186k              │ own finds : 807       │
│  dictionary : 0/0, 0/0, 0/0, 0/0                   │  imported : 0         │
│havoc/splice : 754/2.13M, 0/0                       │ stability : 100.00%   │
│py/custom/rq : unused, unused, unused, unused       ├───────────────────────┘
│    trim/eff : 7.28%/197k, 99.28%                   │          [cpu000: 75%]
└─ strategy: explore ────────── state: in progress ──┘^C

+++ Testing aborted by user +++
[*] Writing ../fuzzing/out/default/fastresume.bin ...
[+] fastresume.bin successfully written with 1094386 bytes.
[+] We're done here. Have a nice day!
```

<div class="rtl-content" markdown="1">

نشرح الداشبورد الخاصة ب AFL++:


1. توقيت العملية (process timing):
   - وقت التشغيل (run time): هاد الوقت يلي اشتغل فيه الفازينغ (ساعة وحدة، و58 دقيقة، و3 ثواني)
   - آخر اكتشاف جديد (last new find): متى آخر مرة لقى طريق جديد (من 20 ثانية)
   - آخر كراش محفوظ (last saved crash): متى آخر مرة انحفظ كراش (من 43 دقيقة و56 ثانية)
   - آخر هانغ محفوظ (last saved hang): متى آخر مرة علق البرنامج وانحفظ (من ساعة و18 دقيقة و42 ثانية)

2. النتائج الكلية (overall results):
   - الدورات المكتملة (cycles done): قديش دورة خلصت (8 دورات)
   - عدد الكوربس (corpus count): عدد ملفات الاختبار الفريدة (810 ملف)
   - الكراشات المحفوظة (saved crashes): عدد الكراشات المحفوظة (6 كراشات)
   - الهانغات المحفوظة (saved hangs): عدد مرات التعليق المحفوظة (4 هانغات)

3. تغطية الماب (map coverage):
   - كثافة الماب (map density): كثافة خريطة التغطية (0.65% / 1.66%)
   - تغطية العد (count coverage): تغطية العد (5.35 بت/تابل)

4. النتائج بعمق (findings in depth):
   - العناصر المفضلة (favored items): العناصر المفضلة (88 أو 10.86%)
   - إيدجس جديدة على (new edges on): حالات الاختبار يلي لقت إيدجس جديدة (161 أو 19.88%)
   - مجموع الكراشات (total crashes): كل الكراشات (30، مع 6 محفوظين)
   - مجموع التايم آوت (total tmouts): كل مرات انتهاء المهلة (881، ما في ولا وحدة محفوظة)

5. نتائج استراتيجية الفازينغ (fuzzing strategy yields): معلومات عن استراتيجيات الفازينغ المختلفة ومدى فعاليتها

6. هندسة العناصر (item geometry):
   - المستويات (levels): عدد المستويات بشجرة الفازينغ (22)
   - العالقة (pending): الحالات العالقة (12)
   - الاكتشافات الخاصة (own finds): الاكتشافات الفريدة (807)
   - الاستقرار (stability): استقرار البرنامج (100.00%)

</div>

## Analyzing the Crashes

<div class="rtl-content" markdown="1">

متل ما شايفين لقينا اكتر من كراش, ف صار وقت تحليل الكراشات.
نحن هون مارح نعمل تحليل كامل, بس رح نرجع نشغل ال CLI مع الملف الي بسبب الكراش.

تحليل الكراشات مهارة و علم كامل لوحده و ممكن اخصصله مقال لقدام ان شاء الله.

</div>

```bash
# Display discovered crashes
ls -la ../fuzzing/out/default/crashes/

id:000000,sig:06,src:000165,time:177841,execs:100655,op:havoc,rep:12
id:000001,sig:06,src:000197,time:202436,execs:114096,op:havoc,rep:7
id:000002,sig:06,src:000468,time:1301778,execs:523167,op:havoc,rep:1
id:000003,sig:11,src:000496,time:1333953,execs:536229,op:havoc,rep:1
id:000004,sig:06,src:000668,time:3675275,execs:1276870,op:havoc,rep:2
id:000005,sig:11,src:000476,time:4446745,execs:1551079,op:havoc,rep:2
README.txt
```

```bash
# Reproduce a specific crash using ASAN for more detailed information
ASAN_OPTIONS=detect_leaks=1:symbolize=1:abort_on_error=1 ./lci ../fuzzing/out/default/crashes/id\:000000\,sig\:06\,src\:000165\,time\:177841\,execs\:100655\,op\:havoc\,rep\:12

=================================================================
==10737==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x515000000500 at pc 0x604822ef6477 bp 0x7ffc0e3af610 sp 0x7ffc0e3af600
READ of size 1 at 0x515000000500 thread T0
    #0 0x604822ef6476 in scanBuffer /home/u24/lci-fuzz/lci/lexer.c:305
    #1 0x604822ef761a in main /home/u24/lci-fuzz/lci/main.c:232
    #2 0x75ad62a2a1c9 in __libc_start_call_main ../sysdeps/nptl/libc_start_call_main.h:58
    #3 0x75ad62a2a28a in __libc_start_main_impl ../csu/libc-start.c:360
    #4 0x604822dc6434 in _start (/home/u24/lci-fuzz/lci/lci+0x20e434) (BuildId: d34ceffc1e31b5aaab680a817b792e4edcfc49eb)

0x515000000500 is located 0 bytes after 512-byte region [0x515000000300,0x515000000500)
allocated by thread T0 here:
    #0 0x604822e7d2e0 in __interceptor_realloc.part.0 (/home/u24/lci-fuzz/lci/lci+0x2c52e0) (BuildId: d34ceffc1e31b5aaab680a817b792e4edcfc49eb)
    #1 0x604822ef70c7 in main /home/u24/lci-fuzz/lci/main.c:195
    #2 0x75ad62a2a1c9 in __libc_start_call_main ../sysdeps/nptl/libc_start_call_main.h:58
    #3 0x75ad62a2a28a in __libc_start_main_impl ../csu/libc-start.c:360
    #4 0x604822dc6434 in _start (/home/u24/lci-fuzz/lci/lci+0x20e434) (BuildId: d34ceffc1e31b5aaab680a817b792e4edcfc49eb)

SUMMARY: AddressSanitizer: heap-buffer-overflow /home/u24/lci-fuzz/lci/lexer.c:305 in scanBuffer
Shadow bytes around the buggy address:
  0x515000000280: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x515000000300: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x515000000380: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x515000000400: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x515000000480: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
=>0x515000000500:[fa]fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x515000000580: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x515000000600: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x515000000680: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x515000000700: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x515000000780: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07
  Heap left redzone:       fa
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
  Left alloca redzone:     ca
  Right alloca redzone:    cb
==10737==ABORTING
Aborted (core dumped)
```

<div class="rtl-content" markdown="1">

طبعا هي الثغرة عبارة عن Heap Buffer Overflow.

ملخص الي مرينا عليه و اتعلمناه هون:

المقال يشرح عملية الفازينغ (Fuzzing) على مجمع (compiler) لغة LOLCODE باستخدام أداة AFL++.

1. **مقدمة عن الفازينغ وأدواته**: استعراض لأشهر أدوات الفازينغ مثل AFL++، libFuzzer، Honggfuzz وغيرها.

2. **تثبيت وإعداد AFL++**: شرح خطوات تثبيت الأداة من GitHub وإعداد البيئة.

3. **تجميع LOLCODE مع AFL++ و ASAN**: شرح كيفية استخدام أداة AFL++ مع Address Sanitizer (ASAN) لاكتشاف مشاكل الذاكرة. يتضمن تعيين متغيرات البيئة وتجميع البرنامج.

4. **إنشاء البذور/المدخلات (Seeds)**: إنشاء أربعة برامج بسيطة بلغة LOLCODE كمدخلات أولية للفازر.

5. **تشغيل الفازر**: شرح كيفية إعداد وتشغيل عملية الفازينغ مع تفسير الخيارات المستخدمة.

6. **تفسير نتائج الفازينغ**: شرح تفصيلي للمعلومات في لوحة تحكم AFL++، بما في ذلك وقت التشغيل والكراشات والتغطية.

7. **تحليل الأخطاء**: عرض كيفية تحليل الكراشات المكتشفة، مع مثال لخطأ heap-buffer-overflow تم اكتشافه في الكود.

الهدف النهائي هو استخدام الفازينغ لاكتشاف الثغرات الأمنية ومشاكل الذاكرة في برامج حقيقية بطريقة عملية.

هذا والله أعلم 💚

</div>