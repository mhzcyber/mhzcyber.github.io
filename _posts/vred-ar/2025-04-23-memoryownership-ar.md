---
title: "memory ownership & double-free arabic"
classes: wide
header:
  teaser: /assets/images/vredar/memoryownership-ar.jpg
ribbon: red
description: "شرح ملكية الذاكرة في البرمجة و الثغرات التي ممكن ان تنتج عنها"
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

# memory ownership & double-free arabic

{: .text-center}

<div class="rtl-content" markdown="1">

## مقدمة

و انا بعمل vulnerability research على برنامج بقرا و بحرر ملفات متل pdf, epub ... الخ لقيت double-free ممكن توصل ل RCE

البرنامج مكتوب بال C

البرنامج بستخدم مكاتب مختلفة ليتعامل مع كل نوع ملف

## مفهوم Memory Ownership

الموضوع بسبب مبدأ اساسي بالبرمجة يسمى memory ownership

بلغات متل java, python الي فيها garbage collection بتتبع و بدير الذاكرة بشكل اوتوماتيكي, بس بال C هي بتكون مسؤولية المبرمج

ال memory ownership المفروض بتجاوب كم سؤال:

- مين مسؤول عن allocating الميموري؟
- مين كان بيقدر او يعدل الميموري؟
- مين بحدد ايمت الميموري هي ما بقا محتاجينها؟
- مين بحرر الميموري او بعمللها freeing؟

طبعا هي الأسئلة الي بسألها اثناء الريسيرش اصلا, لانو بحال اي خطأ بهدول غالبا في قدامنا:

* double-free
* use-after-free
* memory leaks 
* ممكن Buffer Overflow (BoF)

## أنماط Memory Ownership

ال Memory ownership الها Models او فينا نقول طرق:

### 1-Exclusive Ownership (الملكية الخاصة)

بكل اختصار هي بتعني انو عنا ال component (an object, a function, or a specific pointer) هو لحاله مالك ل مكان معين بالذاكر memory و لحاله المسؤول عن تحرير freeing الميمور هي.

كمثال:
</div>


```c
void function() {
    char *buffer = malloc(100);  // This function owns the buffer
    // ... use buffer ...
    free(buffer);  // The owner is responsible for cleanup
}
```

<div class="rtl-content" markdown="1">

### 2-Transfer Ownership (مشاركة الملكية)
بسهولة انت بتنقل الصلاحية و الملكية من component ل component اخر او من جزء ل جزء اخر بالبرنامج
كمثال:

</div>

```c
// The caller becomes responsible for freeing the returned string
char* create_greeting(const char* name) {
    char* greeting = malloc(strlen(name) + 10);
    sprintf(greeting, "Hello, %s!", name);
    return greeting;  // Ownership is transferred to the caller
}
void main() {
    char* message = create_greeting("Alice");
    printf("%s\n", message);
    free(message);  // Now the caller's responsibility
}
```

<div class="rtl-content" markdown="1">

### 3- Shared ownership - with reference counting 
لما يكون عنا برنامج معقد و اكتر من جزء من البرنامج محتاج يتشارك بنفس ال object, dats structure ...الخ ممكن هي الطريقة تستخدم و بكون معنا عداد reference counting لنعد كم pointer متشارك الملكية.
مثال:

</div>

```c
typedef struct {
    void* data;
    int ref_count;
} SharedResource;
SharedResource* resource_create() {
    SharedResource* res = malloc(sizeof(SharedResource));
    res->data = malloc(1000);
    res->ref_count = 1;
    return res;
}
void resource_acquire(SharedResource* res) {
    res->ref_count++;
}
void resource_release(SharedResource* res) {
    res->ref_count--;
    if (res->ref_count == 0) {
        free(res->data);
        free(res);
    }
}
```


<div class="rtl-content" markdown="1">

### 4- Non-Ownership references
 هي الطبيعية لما جزء من الكود اله وصول للميموري بس بدون صلاحيات او ملكية
مثال:
</div>

```c
void print_data(const char* data) {
    // This function can use 'data' but doesn't own it
    // It should NOT attempt to free this memory
    printf("Data: %s\n", data);
}
```

<div class="rtl-content" markdown="1">

## كيف بتصير الثغرة

ال pattern او النمط البرمجي الي بسبب ال double-free طبعا ال double-free من اسمها بتصير لما نفس الجزء من الذاكرة بنعملله free مرتين.
عنا حالتين من وجهة نظر memory ownership بصير فيها الموضوع هاد

### 1- الملكية مالها واضحة لمين

يعني الكود مكتوب بطريقة ما موضح فيها اي جزء من البرنامج بملك الميموري بالتالي بصير غلط و جزئين او 2 components بالبرنامج بعملو free لنفس الميموري.
مثال:

</div>

```c
void problematic_function(char* shared_buffer) {
    // Is this function supposed to free shared_buffer?
    // The caller doesn't know.
    free(shared_buffer);  // First free
}
void main() {
    char* buffer = malloc(100);
    problematic_function(buffer);
    free(buffer);  // Second free - CRASH!
}
```
<div class="rtl-content" markdown="1">

### 2- مشكلة ownership transfer 

لما ال ownership transfer ماله متبرمج بشكل صح
مثال:
</div>

```c
// Poor documentation doesn't specify if caller now owns the memory
char* get_config_path() {
    return strdup("/etc/app/config");  // Creates a new string
}
void main() {
    char* path = get_config_path();
    // Should I free this? The documentation doesn't say!
    use_path(path);
    free(path);  // May be correct or may be wrong
}
```

<div class="rtl-content" markdown="1">

## نمط الثغرة

النمط الي انا لقيته و سبب الثغرة كان يعتبر بسيط
- عنا مكتبة بتتعامل مع احد انواع الملفات
- طبيعي المكتبة بتملك objects و غيره من ال data structure بالعموم
- ال function الي من المكتبة بترجع return ل pointers خاصين بال objects تبعيت المكتبة بدون ما تنقل الملكية , بالتالي المكتبة لسا المالكة للميموري.
- هون الخطأ, البرنامج الي بستخدم المكتبة, المبرمج يبدو ما انتبه لل library documentation و ماعرف مين بملك شو و شو مسؤولية شو و شو منشان شو (إنها اللهجة السورية أيها السادة)
- البرنامج بعمل freeing للميموري
- المكتبة بترجع بتعمل free للميموري بالتالي بصير double-free.

</div>