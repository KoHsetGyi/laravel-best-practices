![Laravel best practices](/images/logo-english.png?raw=true)

Translations:

[Русский](russian.md)



It's not a Laravel adaptation of SOLID principles, patterns etc. Here you'll find the best practices which are usually ignored in real life Laravel projects.

## Contents

[Single responsibility principle](#single-responsibility-principle)

[Fat models, skinny controllers](#fat-models-skinny-controllers)

[Validation](#validation)

[Business logic should be in service class](#business-logic-should-be-in-service-class)

[Don't repeat yourself (DRY)](#dont-repeat-yourself-dry)

[Prefer to use Eloquent over using Query Builder and raw SQL queries. Prefer collections over arrays](#prefer-to-use-eloquent-over-using-query-builder-and-raw-sql-queries-prefer-collections-over-arrays)

[Mass assignment](#mass-assignment)

[Do not execute queries in Blade templates and use eager loading (N + 1 problem)](#do-not-execute-queries-in-blade-templates-and-use-eager-loading-n--1-problem)

[Comment your code, but prefer descriptive method and variable names over comments](#comment-your-code-but-prefer-descriptive-method-and-variable-names-over-comments)

[Do not put JS and CSS in Blade templates and do not put any HTML in PHP classes](#do-not-put-js-and-css-in-blade-templates-and-do-not-put-any-html-in-php-classes)

[Use config and language files, constants instead of text in the code](#use-config-and-language-files-constants-instead-of-text-in-the-code)

[Use standard Laravel tools accepted by community](#use-standard-laravel-tools-accepted-by-community)

[Follow Laravel naming conventions](#follow-laravel-naming-conventions)

[Use shorter and more readable syntax where possible](#use-shorter-and-more-readable-syntax-where-possible)

[Use IoC container or facades instead of new Class](#use-ioc-container-or-facades-instead-of-new-class)

[Other good practices](#other-good-practices)

### **Single responsibility principle**

A class and a method should have only one responsibility.

Bad:

```
public function getFullNameAttribute()
{
    if (auth()->user() && auth()->user()->hasRole('client') && auth()->user()->isVerified()) {
        return 'Mr. ' . $this->first_name . ' ' . $this->middle_name . ' ' $this->last_name;
    } else {
        return $this->first_name[0] . '. ' . $this->last_name;
    }
}
```

Good:

```
public function getFullNameAttribute()
{
    return $this->isVerifiedClient() ? $this->getFullNameLong() : $this->getFullNameShort();
}

public function isVerfiedClient()
{
    return auth()->user() && auth()->user()->hasRole('client') && auth()->user()->isVerified();
}

public function getFullNameLong()
{
    return 'Mr. ' . $this->first_name . ' ' . $this->middle_name . ' ' . $this->last_name;
}

public function getFullNameShort()
{
    return $this->first_name[0] . '. ' . $this->last_name;
}
```

[🔝 Back to contents](#contents)

### **Fat models, skinny controllers**

Put all DB related logic into Eloquent models or into Repository classes if you're using Query Builder or raw SQL queries.

Bad:

```
public function index()
{
    $clients = Client::verified()
        ->with(['orders' => function ($q) {
            $q->where('created_at', '>', Carbon::today()->subWeek());
        }])
        ->get();

    return view('index', ['clients' => $clients]);
}
```

Good:

```
public function index()
{
    return view('index', ['clients' => $this->client->getWithNewOrders()]);
}

Class Client extends Model
{
    public function getWithNewOrders()
    {
        return $this->verified()
            ->with(['orders' => function ($q) {
                $q->where('created_at', '>', Carbon::today()->subWeek());
            }])
            ->get();
    }
}
```

[🔝 Back to contents](#contents)

### **Validation**

Move validation from controllers to Request classes.

Bad:

```
public function store(Request $request)
{
    $request->validate([
        'title' => 'required|unique:posts|max:255',
        'body' => 'required',
        'publish_at' => 'nullable|date',
    ]);

    ....
}
```

Good:

```
public function store(PostRequest $request)
{    
    ....
}

class PostRequest extends Request
{
    public function rules()
    {
        return [
            'title' => 'required|unique:posts|max:255',
            'body' => 'required',
            'publish_at' => 'nullable|date',
        ];
    }
}
```

[🔝 Back to contents](#contents)

### **Business logic should be in service class**

A controller must have only one responsibility, so move business logic from controllers to service classes.

Bad:

```
public function store(Request $request)
{
    if ($request->hasFile('image')) {
        $request->file('image')->move(public_path('images') . 'temp');
    }
    
    ....
}
```

Good:

```
public function store(Request $request)
{
    $this->articleService->handleUploadedImage($request->file('image'));

    ....
}

class ArticleService
{
    public function handleUploadedImage($image)
    {
        if (!is_null($image)) {
            $image->move(public_path('images') . 'temp');
        }
    }
}
```

[🔝 Back to contents](#contents)

### **Don't repeat yourself (DRY)**

Reuse code when you can. SRP is helping you to avoid duplication. Also, reuse Blade templates, use Eloquent scopes etc.

Bad:

```
public function getActive()
{
    return $this->where('verified', 1)->whereNotNull('deleted_at')->get();
}

public function getArticles()
{
    return $this->whereHas('user', function ($q) {
            $q->where('verified', 1)->whereNotNull('deleted_at');
        })->get();
}
```

Good:

```
public function scopeActive($q)
{
    return $q->where('verified', 1)->whereNotNull('deleted_at');
}

public function getActive()
{
    return $this->active()->get();
}

public function getArticles()
{
    return $this->whereHas('user', function ($q) {
            $q->active();
        })->get();
}
```

[🔝 Back to contents](#contents)

### **Prefer to use Eloquent over using Query Builder and raw SQL queries. Prefer collections over arrays**

Eloquent allows you to write readable and maintainable code. Also, Eloquent has great built-in tools like soft deletes, events, scopes etc.

Bad:

```
SELECT *
FROM `articles`
WHERE EXISTS (SELECT *
              FROM `users`
              WHERE `articles`.`user_id` = `users`.`id`
              AND EXISTS (SELECT *
                          FROM `profiles`
                          WHERE `profiles`.`user_id` = `users`.`id`) 
              AND `users`.`deleted_at` IS NULL)
AND `verified` = '1'
AND `active` = '1'
ORDER BY `created_at` DESC
```

Good:

```
Article::has('user.profile')->verified()->latest()->get();
```

[🔝 Back to contents](#contents)

### **Mass assignment**

Bad:

```
$article = new Article;
$article->title = $request->title;
$article->content = $request->content;
$article->verified = $request->verified;
// Привязать статью к категории.
$article->category_id = $category->id;
$article->save();
```

Good:

```
$category->article()->create($request->all());
```

[🔝 Back to contents](#contents)

### **Do not execute queries in Blade templates and use eager loading (N + 1 problem)**

Bad (for 100 users, 101 DB queries will be executed):

```
@foreach (User::all() as $user)
    {{ $user->profile->name }}
@endforeach
```

Good (for 100 users, 2 DB queries will be executed):

```
$users = User::with('profile')->get();

...

@foreach ($users as $user)
    {{ $user->profile->name }}
@endforeach
```

[🔝 Back to contents](#contents)

### **Comment your code, but prefer descriptive method and variable names over comments**

Bad:

```
if (count((array) $builder->getQuery()->joins) > 0)
```

Better:

```
// Determine if there are any joins.
if (count((array) $builder->getQuery()->joins) > 0)
```

Good:

```
if ($this->hasJoins())
```

[🔝 Back to contents](#contents)

### **Do not put JS and CSS in Blade templates and do not put any HTML in PHP classes**

Bad:

```
let article = `{{ json_encode($article) }}`;
```

Better:

```
<input id="article" type="hidden" value="{{ json_encode($article) }}">

Or

<button class="js-fav-article" data-article="{{ json_encode($article) }}">{{ $article->name }}<button>
```

В Javascript файле:

```
let article = $('#article').val();
```

The best way is to use specialized PHP to JS package to transfer the data.

[🔝 Back to contents](#contents)

### **Use config and language files, constants instead of text in the code**

Bad:

```
public function isNormal()
{
    return $article->type === 'normal';
}

return back()->with('message', 'Your article has been added!');
```

Good:

```
public function isNormal()
{
    return $article->type === Article::TYPE_NORMAL;
}

return back()->with('message', __('app.article_added'));
```

[🔝 Back to contents](#contents)

### **Use standard Laravel tools accepted by community**

Prefer to use built-in Laravel functionality and community packages instead of using 3rd party packages and tools. Any developer who will work with your app in the future will need to learn new tools. Also, chances to get help from the Laravel community are significantly lower when you're using a 3rd party package or tool. Do not make your client pay for that.

Task | Standard tools | 3rd party tools
------------ | ------------- | -------------
Authorization | Policies | Entrust, Sentinel and other packages
Compiling assets | Laravel Mix | Grunt, Gulp, 3rd party packages
Development Environment | Homestead | Docker
Deployment | Laravel Forge | Deployer and other solutions
Unit testing | PHPUnit, Mockery | Phpspec
Browser testing | Laravel Dusk | Codeception
DB | Eloquent | SQL, Doctrine
Templates | Blade | Twig
Working with data | Laravel collections | Arrays
Form validation | Request classes | 3rd party packages, validation in controller
Authentication | Built-in | 3rd party packages, your own solution
API authentication | Laravel Passport | 3rd party JWT and OAuth packages
Creating API | Built-in | Dingo API and similar packages
Working with DB structure | Migrations | Working with DB structure directly
Localization | Built-in | 3rd party packages
Realtime user interfaces | Laravel Echo, Pusher | 3rd party packages and working with WebSockets directly
Generating testing data | Seeder classes, Model Factories, Faker | Creating testing data manually
Task scheduling | Laravel Task Scheduler | Scripts and 3rd party packages
DB | MySQL, PostgreSQL, SQLite, SQL Server | MongoDB

[🔝 Back to contents](#contents)

### **Follow Laravel naming conventions**

 Follow [PSR standards](http://www.php-fig.org/psr/psr-2/).
 
 Also, follow naming conventions accepted by Laravel community:

What | How | Good | Bad
------------ | ------------- | ------------- | -------------
Controller | singular | ArticleController | ~~ArticlesController~~
Route | plural | articles/1 | ~~article/1~~
Named route | snake_case with dot notation | users.show_active | ~~users.show-active, show-active-users~~
Model | singular | User | ~~Users~~
hasOne or belongsTo relationship | singular | articleComment | ~~articleComments, article_comment~~
All other relationships | plural | articleComments | ~~articleComment, article_comments~~
Table | plural | article_comments | ~~article_comment, articleComments~~
Pivot table | singular model names in alphabetical order | article_user | ~~user_article, articles_users~~
Table column | snake_case without model name | meta_title | ~~MetaTitle; article_meta_title~~
Foreign key | singular model name with _id suffix | article_id | ~~ArticleId, id_article, articles_id~~
Primary key | - | id | ~~custom_id~~
Migration | - | 2017_01_01_000000_create_articles_table | ~~2017_01_01_000000_articles~~
Method | camelCase | getAll | ~~get_all~~
Method in resource controller | [table](https://laravel.com/docs/master/controllers#resource-controllers) | store | ~~saveArticle~~
Method in test class | camelCase | testGuestCannotSeeArticle | ~~test_guest_cannot_see_article~~
Variable | camelCase | $articlesWithAuthor | ~~$articles_with_author~~
Collection | descriptive, plural | $activeUsers = User::active()->get() | ~~$active, $data~~
Object | descriptive, singular | $activeUser = User::active()->first() | ~~$users, $obj~~
Config and language files index | snake_case | articles_enabled | ~~ArticlesEnabled; articles-enabled~~
View | snake_case | show_filtered.blade.php | ~~showFiltered.blade.php, show-filtered.blade.php~~
Config | snake_case | google_calendar.php | ~~googleCalendar.php, google-calendar.php~~
Contract (interface) | adjective or noun | Authenticatable | ~~AuthenticationInterface, IAuthentication~~
Trait | adjective | Notifiable | ~~NotificationTrait~~

[🔝 Back to contents](#contents)

### **Use shorter and more readable syntax where possible**

Bad:

```
$request->session()->get('cart');
$request->input('name');
```

Good:

```
session('cart');
$request->name;
```

More examples:

Common syntax | Shorter and more readable syntax
------------ | -------------
`Session::get('cart')` | `session('cart')`
`$request->session()->get('cart')` | `session('cart')`
`Session::put('cart', $data)` | `session(['cart' => $data])`
`$request->input('name')` | `$request->name`
`Request::get('name')` | `request('name')`
`return Redirect::back()` | `return back()`
`return view('index')->with('title', $title)->with('client', $client)` | `return view('index', compact('title', 'client'))`

[🔝 Back to contents](#contents)

### **Use IoC container or facades instead of new Class**

new Class syntax creates tight coupling between classes and complicates testing. Use IoC container or facades instead.

Bad:

```
$user = new User;
$user->create($request->all());
```

Good:

```
public function __construct(User $user)
{
    $this->user = $user;
}

....

$this->user->create($request->all());
```

[🔝 Back to contents](#contents)

### **Other good practices**

Never put any logic in routes files.

Try not to use vanilla PHP in Blade templates.

[🔝 Back to contents](#contents)