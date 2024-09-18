---
title: "How I Reduced 16k Queries Per Day"
date: 2024-09-18
tags: Laravel, Eloquent, code architecture
description: How an Eloquent method caused many unneeded queries and how we tracked it down
---

On my team, the engineer who handles our devops had noted several times that we were seeing spikes in database latency and queries. The periodicity lined up with a command we ran every 5 minutes to scan for updates and then sync updated records with a vendor. I figured the solution was to maybe run the command less frequently, but because it was simply a paper cut, we had not allotted any time to fixing it.

### Investigation

Looking at the MySQL queries in the PDO service of DataDog, I observed that he was indeed correct. Every five minutes, latency spiked, along with the number of queries. I decided to do some investigation.

After looking at the command and then the jobs that the command dispatched, I noticed there were 16,000 queries per day for `select distinct(commentable_type) from comments`. I scoured the code related to the command and jobs, but saw no use of `distinct` at all.

### Further Investigation

I ran the command locally using the `DB::pretend()` function. This function allows us to observe the database queries that would be executed without them actually running.

```php
\DB::enableQueryLog();
\DB::pretend(\Artisan::call('comments:scan-for-updates'));
dump(\DB::getRawQueryLog());
```

Sure enough, even with no comments records in my local database, there was this call to fetch distinct `commentable_type` entries. It definitely wasn't in our code... where was this happening?

### Bring on XDebug

XDebug has been the most valuable tool for learning the inner workings of Laravel.  While some people think that `dd()` is all you need, I have found that my understanding exploded once I got familiar with step debugging through code.

### Some Background

Laravel's Eloquent has the handy ability to check for the existence of a relation and eager-load it, all in one fell swoop using [Builder::withWhereHas()](https://laravel.com/docs/11.x/eloquent-relationships#constraining-eager-loads-with-relationship-existence).

For instance, say we have a few models.

```php
namespace App\Models;
 
class User extends Model
{
}

class Comment extends Model
{
    /**
     * Get the parent commentable model (post or video).
     */
    public function commentable(): MorphTo
    {
        return $this->morphTo();
    }
}

 
class Post extends Model
{
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    /**
     * Get all of the post's comments.
     */
    public function comments(): MorphMany
    {
        return $this->morphMany(Comment::class, 'commentable');
    }
}

class Video extends Model
{
    /**
     * Get all of the video's comments.
     */
    public function comments(): MorphMany
    {
        return $this->morphMany(Comment::class, 'commentable');
    }
}
```

We can now query for all of the posts that have users and eager-load that relationship in a single call. For instance, maybe we want to load all of the posts which have a user and send them a message.

```php
$posts = Post::query()
    ->withWhereHas('user')
    ->where('created_at', '>=', now()->subMinutes(5))
    ->get();

foreach($posts as $post) {
    $posts->user->notify(new ThanksForCommentingMessage($post));
}
```

### The Problem

In our code to scan for updates, we were leveraging `withWhereHas`.

```php
Comment::query()
    ->withWhereHas('commentable')
    ->chunkById(250, function(Collection $comments) {
        foreach($comments as $comment) {
            ThirdPartyService::sendUpdate($comment);
        }
    }
);
```

Using `withWhereHas` on a morph relationship turns out to be a very expensive convenience. Why?

Looking [at the method itself](https://github.com/laravel/framework/blob/0d0f55f1ea7046a55cb93b2c8e35d856c5a35d59/src/Illuminate/Database/Eloquent/Concerns/QueriesRelationships.php#L167-L171), it doesn't seem obvious why it would cause so much pain. If we jump three calls deeper at `QueriesRelationships::hasMorph()`, we can see the problem.

```php
public function hasMorph($relation, $types, $operator = '>=', $count = 1, $boolean = 'and', ?Closure $callback = null)
{
    if (is_string($relation)) {
        $relation = $this->getRelationWithoutConstraints($relation);
    }

    $types = (array) $types;
    if ($types === ['*']) {
        $types = $this->model->newModelQuery()->distinct()->pluck($relation->getMorphType())->filter()->all();
    }

    // ... omitted for brevity
}
```

Through the chain of calls, `$types` is always `['*']`. Without explicitly passing the types, Laravel makes a query to find all of the morphable types via `select distinct(commentable_type) from comments`. Now that Laravel knows the possible types, it can build a query to check for existence in those related tables.

### The Solution

We only morph to a few select types, so having to perform a query to see the possible morphable_types is unnecessary. We were able to simply change our code to this:

```php
Comment::query()
    ->whereHasMorph(
        'commentable',
        [Post::class, Video::class] // the only classes we morph to
    )
    ->chunkById(250, function(Collection $comments) {
        $comments->load('commentable');
        foreach($comments as $comment) {
            UpdateThirdParty::sendUpdate($comment);
        }
    }
);
```

No more distinct query! Plus we still have lazy-loading for the `commentable` models.

## Conclusion

### Hol up! The math ain't mathing

If you run this command every five minutes, how did we get 16k queries per day? The scheduled command fans out, and the fan out jobs are what actually trigger this query.

### The Real Conclusion

Something I heard today that I think really hits the mark: "There is no such thing as magic, just work that is being done elsewhere."  The convenience of using a framework comes at the cost of oddities like this. Profiling, debugging, and observability tools allow us to peel back the curtain and see the magic for ourselves. Understanding a problem through observation and deducing the root cause are key skills that make a software developer truly valuable.

---

Want to come work on Laravel stuff with me? [Teamworks is hiring](https://ats.rippling.com/teamworks-careers/jobs/5bcdf8ec-a842-44ce-aecd-bdebdcf592c6).
