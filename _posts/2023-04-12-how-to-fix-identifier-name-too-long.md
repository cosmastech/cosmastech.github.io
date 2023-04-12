---
title: "How to fix the 'Identifier name too long' exception"
date: 2023-04-12
tags: Laravel, database, mysql
---

Do you love verbosity in your database tables and columns? ✔️

Are you running into issues where Laravel's migration helpers like `foreignIdFor()` are not letting your migrations run? ✖️

Take for instance this sample migration:

```php
use App\Models\KnowledgeBaseResource;
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;


return new class extends Migration
{
    public function up(): void
    {
        Schema::create('company_knowledge_base_resource_matches', function (Blueprint $table) {
            $table->id();
            $table->foreignIdFor(KnowledgeBaseResource::class)
                ->nullable()
                ->constrained()
                ->nullOnDelete();
            $table->timestamps();
        });
    }
}
```

When I do run `php artisan migrate`, I get the following:

```
SQLSTATE[42000]: Syntax error or access violation: 1059 Identifier name 'company_knowledge_base_resource_matches_knowledge_resource_id_foreign' is too long
(Connection: mysql, SQL: alter table `company_knowledge_base_resource_matches` add constraint `company_knowledge_base_resource_matches_knowledge_resource_id_foreign` foreign key (`knowledge_base_resource_id`) references `knowledge_base_resources` (`id`) on delete set null)
```

Extra frustrating because the table was created successfully, but it's just adding the foreign key constraint that fails.

In the past, the workaround was to have to write the foreign key relationship manually

```php
Schema::create('company_knowledge_base_resource_matches', function (Blueprint $table) {
    $table->id();
    $table->foreignIdFor(KnowledgeBaseResource::class)->nullable();
    $table->timestamps();
    
    $table->foreign('knowledge_base_resource_id', 'my_foreign_index')
        ->references('id')
        ->on('knowledge_base_resources')
        ->constrained()
        ->nullOnDelete();
});
```

This leaves more room for error and is just a little bit ugly.

With (this PR)[https://github.com/laravel/framework/pull/46746] you can now specify the index name when calling `constrained()`
```php
Schema::create('company_knowledge_base_resource_matches', function (Blueprint $table) {
    $table->id();
    $table->foreignIdFor(KnowledgeBaseResource::class)
        ->nullable()
        ->constrained(indexName: 'my_foreign_index')
        ->nullOnDelete();
    $table->timestamps();
});
```

Voila! Now your index name will be `my_foreign_index` and the migration runs without exception. Of course, if you love verbosity and precision like I do, you'll probably choose a better index name than that :laughing:
