![Plastic Logo](http://i.imgur.com/PyolY7g.png)

> Plastic is an Elasticsearch ODM and mapper for Laravel.
> It renders the developer experience more enjoyable while using Elasticsearch, by providing a fluent syntax for mapping, querying, and storing eloquent models.

[![License](https://poser.pugx.org/laravel/framework/license.svg)](https://packagist.org/packages/sleimanx2/plastic)
[![Build Status](https://travis-ci.org/sleimanx2/plastic.svg?branch=master&&refresh=2)](https://travis-ci.org/sleimanx2/plastic)
[![StyleCI](https://styleci.io/repos/58264395/shield)](https://styleci.io/repos/58264395)

> This package is still under active development and may change.

## Installing Plastic

```bash
composer require sleimanx2/plastic
```

Then we need to add the plastic service provider to `config/app.php` under the providers key:

```php
Sleimanx2\Plastic\PlasticServiceProvider
```

Finally we need to run:

```bash
php artisan vendor:publish
```

This will create a config file at `config/plastic.php` and a mapping directory at `database/mappings`.

## Usage
- [Defining Searchable Models](#searchable-models)
- [Storing Model Content](#store-content)
- [Searching](#searching)
- [Aggregation](#aggregation)
- [Suggestions](#suggestions)
- [Mappings](#mappings)


### <a name="searchable-models">Defining Searchable Models</a>

To get started, enable searching capabilities in your model by adding the `Sleimanx2\Plastic\Searchable` trait:

```php
use Sleimanx2\Plastic\Searchable;

class Book extends Model
{
    use Searchable;
}
```

#### Defining what data to store.

By default, Plastic will store all visible properties of your model, using `$model->toArray()`.

In addition, Plastic provides you with two ways to manually specify which attributes/relations should be stored in Elasticsearch.

##### 1 — Providing a searchable property to our model

```php
public $searchable = ['id', 'name', 'body', 'tags', 'images'];
```

##### 2 — Providing a buildDocument method

```php
public function buildDocument()
{
    return [
        'id' => $this->id,
        'tags' => $this->tags
    ];
}
```

#### Custom elastic type name

By the default Plastic will use the model table name as the model type. You can customize it by adding a `$documentType` property to your model:

```php
public $documentType = 'custom_type';
```

### <a name="store-content">Storing Model Content</a>

Plastic automatically syncs model data with elastic when you save or delete your model from our SQL DB, however this feature can be disabled by adding `public $syncDocument = false` to your model.

> Its important to note that manual document update should be performed in multiple scenarios:
>
> 1 - When you perform a bulk update or delete, no Eloquent event is triggered, therefore the document data won't be synced.
>
> 2 - Plastic doesn't listen to related models events (yet), so when you update a related model’s content you should consider updating the parent document.

#### Saving a document

```php
$book = Book::first()->document()->save();
```

#### Partial updating a document

```php
$book = Book::first()->document()->update();
```

#### Deleting a document

```php
$book = Book::first()->document()->delete();
```

#### Saving documents in bulk

```php
$tag = Tag::first();

$tag->document()->bulkSave($tag->books);
```

#### Deleting documents in bulk

```php
$author = Author::first();

$author->document()->bulkDelete($author->books);
```

#### Reindexing documents in bulk

```php
$post = new Post();

$post->document()->reindex($post->all());
```

### <a name="searching">Searching Model Content</a>

Plastic provides a fluent syntax to query Elasticsearch which leads to compact readable code. Lets dig into it:

```php
$result = Book::search()->match('title','pulp')->get();

// Returns a collection of Book Models
$books = $result->hits();

// Returns the total number of matched documents
$result->totalHits();

// Returns the highest query score
$result->maxScore();

//Returns the time needed to execute the query
$result->took();
```

To get the raw DSL query that will be executed you can call `toDSL()`:

```php
$dsl = Book::search()->match('title','pulp')->toDSL();
```

#### Pagination

```php
$books = Book::search()
    ->multiMatch(['title', 'description'], 'ham on rye', ['fuzziness' => 'AUTO'])
    ->orderBy('date')
    ->paginate();
```

You can still access the result object after pagination using the result method:

```php
$books->result();
```

#### Bool Query

```php
User::search()
    ->must()
        ->term('name','kimchy')
    ->mustNot()
        ->range('age',['from'=>10,'to'=>20]);
    ->should()
        ->match('bio','developer')
        ->match('bio','elastic')
    ->filter()
        ->term('tag','tech')
    ->get();
```

#### Nested Query

```php
$contain = 'foo';

Post::search()
    ->multiMatch(['title', 'body'], $contain)
    ->nested('tags', function (SearchBuilder $builder) use ($contain) {
        $builder->match('tags.name', $contain);
    });
```

> Check out this [documentation](docs/search.md) of supported search queries within Plastic and how to apply unsupported queries.

### <a name="aggregation">Aggregation</a>

```php
$result = User::search()
    ->match('bio', 'elastic')
    ->aggregate(function (AggregationBuilder $builder) {
        $builder->average('average_age', 'age');
    });

$aggregations = $result->aggregations();
```

> Check out this [documentation](docs/aggregation.md) of supported aggregations within plastic and how to apply unsupported aggregations.

### <a name="suggestions">Suggestions</a>

```php
Plastic::suggest()->completion('tag_suggest', 'photo')->get();
```

The suggestions query builder can also be accessed directly from the model as well:

```php
Tag::suggest()->term('tag_term','admin')->get();
```

### <a name="mappings">Model Mapping</a>

Mappings are an important aspect of Elasticsearch. You can compare them to indexing in SQL databases. Mapping your models yields better and more efficient search results, and allows us to use some special query functions like nested fields and suggestions.

#### Generate a Model Mapping

```bash
php artisan make:mapping "App\User"
```

The new mapping will be placed in your `database/mappings` directory.

#### Mapping Structure

A mapping class contains a single method `map`. The map method is used to map the given model fields.

Within the `map` method you may use the Plastic Map builder to expressively create field maps. For example, let's look at a sample mapping that creates a Tag model map:

```php
use Sleimanx2\Plastic\Map\Blueprint;
use Sleimanx2\Plastic\Mappings\Mapping;

class AppTag extends Mapping
{
    /**
     * Full name of the model that should be mapped
     *
     * @var string
     */
    protected $model = App\Tag::class;

    /**
     * Run the mapping.
     *
     * @return void
     */
    public function map()
    {
        Map::create($this->getModelType(), function (Blueprint $map) {
            $map->string('name')->store('true')->index('analyzed');

            // instead of the fluent syntax we can use the second method argument to fill the attributes
            $map->completion('suggestion', ['analyzer' => 'simple', 'search_analyzer' => 'simple'];
        });
    }
}
```

> To learn about all of the methods available on the Map builder, check out this  [documentation](docs/mapping.md).

#### Run Mappings

Running the created mappings can be done using the Artisan console command:

```bash
php artisan mapping:run
```

#### Updating Mappings

If your update consists only of adding a new field mapping you can always update our model map with your new field and run: 

```bash
php artisan mapping:rerun
```

The mapping for existing fields cannot be updated or deleted, so you’ll need to use one of following techniques to update existing fields.

##### 1 — Create a new index

You can always create a new Elasticsearch index and re-run the mappings.
After running the mappings you can use the `bulkSave` method to sync your SQL data with Elasticsearch.

##### 2 — Using aliases

Its recommended to create your Elasticsearch index with an alias to ease the process of updating your model mappings with zero downtime. To learn more check out:

https://www.elastic.co/blog/changing-mapping-with-zero-downtime

> You can access the Elasticsearch client to manage your indices and aliases using `$client = Plastic::getClient()`

## Contributing

Thank you for contributing, The contribution guide can be found [Here](CONTRIBUTING.md).

## License

Plastic is open-sourced software licensed under the [MIT license](LICENSE.md).

## To Do

### Search Query Builder

 - [ ] implement Boosting query
 - [ ] implement ConstantScore query
 - [ ] implement DisMaxQuery query
 - [ ] implement MoreLikeThis query (with raw eloquent models)
 - [ ] implement GeoShape query

### Aggregation Query Builder

 - [ ] implement Nested aggregation
 - [ ] implement ExtendedStats aggregation
 - [ ] implement TopHits aggregation

### Mapping

 - [ ] Find a seamless way to update field mappings with zero downtime with aliases

### General

 - [ ] Better query builder documentation
