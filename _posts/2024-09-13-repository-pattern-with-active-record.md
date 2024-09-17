---
title: "Laravel People (Generally) Don't Like Repositories"
date: 2024-09-13
tags: ActiveRecord, Laravel, Eloquent, code architecture
description: The inherent pain of ActiveRecord
---

Search [/r/laravel](https://www.reddit.com/r/laravel/) or Twitter for discussions about Laravel and repositories; the general consensus is that repositories are not necessary. Perhaps they are a vestige from a bygone era, or just something that developers from other languages and frameworks use because they don't understand Eloquent.

Laravel's Eloquent implements the [ActiveRecord pattern](https://martinfowler.com/eaaCatalog/activeRecord.html). I think if you're using Laravel to write a quick MVP, or you're working on a small project and/or very small team, Eloquent models are just fine and offer a massive speed boost. The reason the repository pattern is so widely used because most frameworks/ORMs don't use ActiveRecord objects. Instead they return simple objects that represent the database state and are totally inert.

Code architecture philosophies like Uncle Bob's Clean Architecture and hexagonal architecture will encourage you to segregate layers and minimize the responsibilities of any component in your system. Active Record is a completely different paradigm, and the data layer "leaks" into every layer of your code.

## For Instance

Say we have a little online store. Consider a few endpoints like `GET /api/products`, `GET /api/products/{product}`, `PUT /api/products/{product}`, and `DELETE /api/products/{product}`.

That controller might look something like this:

```php
class ProductController extends Controller
{
    public function index(IndexProductsRequest $request): ResourceCollection
    {
        return ProductResource::collection(Product::paginate());
    }

    public function show(
        ShowProductRequest $request,
        Product $product
    ): ProductResource
    {
        $product->product_views->increment();

        return ProductResource::make($product);
    }

    public function update(
        UpdateProductRequest $request,
        Product $product,
        UpdateVariationsAction $updateVariationsAction
    ): ProductResource
    {
        $product->name = (string) $request->string('name');
        $product->price = $request->int('price');
        $product->save();

        $updateVariationsAction->execute(
            $product,
            $request->validated('variations')
        );

        return ProductResource::make($product);
    }

    public function destroy(
        DestroyProductRequest $request,
        Product $product
    ): Response
    {
        $product->delete();

        return response()->noContent();
    }
}
```

Your `UpdateProductRequest` and `DestroyProductRequest` classes will likely have some validation that confirms the requesting user has access to view/modify/delete the product. For instance, the `show` endpoint doesn't show a product until its launch date.

Your `ProductResource` class might return data about tables related to the Product (like variations of the product, which live in a separate table).

## Yes, So Why Is That a Problem?

Your request has access to the Product model, your controller method has access to the Product because Laravel automatically binds it, and your resource has access to the Product model. Any of these can:

* mutate the model's properties but not save it
* fetch other models via relationships
* update the database
* delete the record

What does that `UpdateVariationsAction` do, and what state does it leave the Product in? Has it eager-loaded the variations before updating them, without refreshing the relationships at the end?

Resources are often the place where N+1 issues hide. That `index()` method hasn't eager-loaded anything that it might need to render a response. Say you're out on vacation for a week and someone merges in a change to the resource which loads records from an additional table, but doesn't eager load it on the `index()` endpoint. If you don't have `preventLazyLoading()` enabled, you would probably only catch this N+1 problem through monitoring.

### Testing

Now say you're writing tests for that `index()` method. You have no way of testing the endpoint without seeding data to your database. If your model's per page limit is 100, how do you test pagination without generating 101+ records and storing them in your database? In addition, you need to also seed all of the relationship data. This slows down test execution and requires additional resources. It can be mitigated by using an in-memory database like SQLite, but this adds variance between your test suite and your production environment. I've personally used SQLite very little, so I have no idea what differences there are between the two systems.

In more traditional styles of architecture which do not leverage ActiveRecord, the controller would use a repository to fetch the product data. In so doing, you can replace the repository in your tests with a test double. That test double would return an array of in-memory objects: plain, inert objects (just like your repository would return).

Unfortunately with Laravel, I haven't found a way to make Models work the same way without complicated use of Mockery that leads to brittle tests, especially if they have relationships you need access to.

## Thanks for all the problems, what's the solution?

I don't have a solution for this.

In my experience, as a project grows in complexity and/or contributors, it requires more discipline to use Eloquent Models in ways that are comprehensible, safe, and clean. I imagine there could be ways of writing architecture tests that enforce rules about which parts of the code are allowed to load relationships or query the database, but my guess is that it's nearly impossible, due to all of the magic properties/methods that Models have.

If anyone has any resources on how to build cleaner architecture within the context of Laravel, I would love to know more. Shoot me an email or reach out on [Twitter](https://x.com/cosmastech).
