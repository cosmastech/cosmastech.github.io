---
title: "Data Integrity At Two Layers"
date: 2023-05-02
tags: Laravel, database, mysql
---

My team loves to enforce type/value integrity at not only the application layer, but also the database layer.

Within the application layer, we rely heavily on [Backed Enums](https://www.php.net/manual/en/language.enumerations.backed.php). They raise a ValueError if there is no matching case. As a bonus, a [column can be cast to an enum easily within Laravel models](https://laravel.com/docs/10.x/eloquent-mutators#enum-casting).

But what about the at the database level?  We rely on foreign key constraints.

Say we have a table called `roles`. 

```sql
CREATE TABLE `roles` (
  `id` smallint unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(191) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL,
  PRIMARY KEY (`id`),
  KEY `role_name_index` (`name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

We can associate a user with their hobbies in a `user_hobbies` table.

```sql
CREATE TABLE `users` (
  `id` bigint unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(191) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL,
  `email` varchar(191) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL,
  `role_id` smallint unsigned NOT NULL,
  PRIMARY KEY (`id`),
  KEY `users_role_id_index` (`role_id`),
  CONSTRAINT `users_role_id_foreign` FOREIGN KEY (`role_id`) REFERENCES `roles` (`id`) ON DELETE CASCADE,
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

