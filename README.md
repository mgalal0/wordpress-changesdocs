# إصلاح مشكلة صفحة البلوج والباجينيشن في ووردبريس

## المشكلة الأساسية

المشكلة كانت تعارض بين:
- صفحة ثابتة (page) باسم "Blog"
- نوع محتوى مخصص (custom post type) باسم "blog"
- المسار الافتراضي للمدونة في وردبريس وهو "/blog"

هذا كان يسبب توجيه الزائر للصفحة الرئيسية بدلاً من صفحة البلوج عند تصفح /blog/page/2/.

## التغييرات في ملف functions.php

### 1. تغيير اسم نوع المحتوى المخصص

```php
function register_section_post_types(){
    // قبل التعديل
    // $pages = ["home", "destinations", "blog", "about", "faq", "contact"];
    
    // بعد التعديل
    $pages = ["home", "destinations", "blog_sections", "about", "faq", "contact"];
    
    foreach($pages as $page){
        // باقي الكود...
    }
}
```

### 2. تحديث دالة remove_trash_option

```php
function remove_trash_option($actions, $post) {
    // قبل التعديل
    // $custom_post_types = ["home", "destinations", "blog", "about", "faq", "contact"];
    
    // بعد التعديل
    $custom_post_types = ["home", "destinations", "blog_sections", "about", "faq", "contact"];
    
    if (in_array($post->post_type, $custom_post_types)) {
        // باقي الكود...
    }
    return $actions;
}
```

### 3. إضافة دعم للمتغير pgn في الروابط

```php
function handle_custom_pagination_query_vars($vars) {
    $vars[] = 'paged';
    return $vars;
}
add_filter('query_vars', 'handle_custom_pagination_query_vars');

// إعادة تهيئة الروابط الدائمة عند الحاجة
function flush_rewrite_on_save() {
    global $wp_rewrite;
    $wp_rewrite->flush_rules();
}
add_action('after_switch_theme', 'flush_rewrite_on_save');
```

## التغييرات في ملف page-blog.php

### 1. تحديث استعلام قسم header البلوج

```php
// قبل التعديل
$blog_sections = get_posts(array(
    'post_type' => 'blog',
    'posts_per_page' => -1,
    'orderby' => 'menu_order',
    'order' => 'ASC',
));

// بعد التعديل
$blog_sections = get_posts(array(
    'post_type' => 'blog_sections', // تم تغييره من blog إلى blog_sections
    'posts_per_page' => -1,
    'orderby' => 'menu_order',
    'order' => 'ASC',
));
```

### 2. تغيير نظام الباجينيشن

```php
// قبل التعديل - الحصول على رقم الصفحة من وردبريس
$paged = (get_query_var('paged')) ? get_query_var('paged') : (get_query_var('page') ? get_query_var('page') : 1);

// بعد التعديل - استخدام معلمة URL مخصصة
$current_page = isset($_GET['pgn']) ? intval($_GET['pgn']) : 1;
```

### 3. تعديل استعلام المقالات

```php
// قبل التعديل
$query = new WP_Query(array(
    'post_type'      => 'post', 
    'posts_per_page' => 5,
    'paged'          => $paged, // المتغير القديم
    'orderby'        => 'date',
    'order'          => 'DESC'
));

// بعد التعديل
$query = new WP_Query(array(
    'post_type'      => 'post', 
    'posts_per_page' => 5,
    'paged'          => $current_page, // المتغير الجديد
    'orderby'        => 'date',
    'order'          => 'DESC'
));
```

### 4. استبدال كود الباجينيشن بكود مخصص

```php
// قبل التعديل
$big = 999999999;
$pagination = paginate_links(array(
    'base'      => esc_url(get_pagenum_link(1)) . '%_%',
    'format'    => 'page/%#%/',
    'current'   => max(1, get_query_var('paged', get_query_var('page'))),
    'total'     => $query->max_num_pages,
    'prev_text' => '<span class="px-3 py-1 border border-gray-300 text-gray-500 rounded-md hover:bg-gray-100">&lt;</span>',
    'next_text' => '<span class="px-3 py-1 border border-gray-300 text-gray-500 rounded-md hover:bg-gray-100">&gt;</span>',
    'before_page_number' => '<span class="page-number px-3 py-1 border border-gray-300 text-gray-500 rounded-md hover:bg-gray-100">',
    'after_page_number' => '</span>',
));

echo str_replace('page-numbers current', 'page-numbers current bg-teal-600 text-white rounded-md', $pagination);

// بعد التعديل - كود باجينيشن مخصص
$total_pages = $query->max_num_pages;

if ($total_pages > 1) {
    // زر السابق
    if ($current_page > 1) {
        echo '<a href="?pgn=' . ($current_page - 1) . '" class="page-numbers"><span class="px-3 py-1 border border-gray-300 text-gray-500 rounded-md hover:bg-gray-100">&lt;</span></a>';
    }
    
    // أرقام الصفحات
    for ($i = 1; $i <= $total_pages; $i++) {
        if ($i == $current_page) {
            echo '<span class="page-numbers current"><span class="page-number px-3 py-1 border border-gray-300 bg-teal-600 text-white rounded-md">' . $i . '</span></span>';
        } else {
            echo '<a href="?pgn=' . $i . '" class="page-numbers"><span class="page-number px-3 py-1 border border-gray-300 text-gray-500 rounded-md hover:bg-gray-100">' . $i . '</span></a>';
        }
    }
    
    // زر التالي
    if ($current_page < $total_pages) {
        echo '<a href="?pgn=' . ($current_page + 1) . '" class="page-numbers"><span class="px-3 py-1 border border-gray-300 text-gray-500 rounded-md hover:bg-gray-100">&gt;</span></a>';
    }
}
```

## الأسباب وراء هذه التغييرات

1. **منع التعارض بين الأسماء**: تغيير اسم نوع المحتوى المخصص من "blog" إلى "blog_sections" يحل مشكلة تعارض الأسماء.

2. **تجنب تعقيدات وردبريس**: استخدام معلمة URL بسيطة (`?pgn=2`) بدلاً من تغيير بنية المسار (`/page/2/`) يتجنب التعارض مع النظام الداخلي لوردبريس.

3. **بساطة التنفيذ**: الحل البسيط يكون أقل عرضة للأخطاء وأسهل في الصيانة.

## كيفية التحقق من أن الإصلاح يعمل

1. تصفح صفحة `/blog` والتأكد من عرض المحتوى بشكل صحيح
2. النقر على رقم الصفحة الثانية ومشاهدة عنوان URL: يجب أن يكون `/blog?pgn=2` 
3. التأكد من عرض محتوى الصفحة الثانية بشكل صحيح بدلاً من التوجيه إلى الصفحة الرئيسية

## خطوات إضافية للتأكد

1. اذهب إلى Settings > Permalinks وانقر على "Save Changes" لإعادة تهيئة الروابط
2. إذا كان لديك plugin للـ cache، قم بمسح الكاش
   
