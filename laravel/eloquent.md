# Eloquent: Getting Started

- [Introduction](#introduction)
- [Retrieving Models](#retrieving-models)
    - [Collections](#collections)
    - [Chunking Results](#chunking-results)
    - [Advanced Subqueries](#advanced-subqueries)
- [Retrieving Single Models / Aggregates](#retrieving-single-models)
    - [Retrieving Aggregates](#retrieving-aggregates)
- [Inserting & Updating Models](#inserting-and-updating-models)
    - [Inserts](#inserts)
    - [Updates](#updates)
    - [Mass Assignment](#mass-assignment)
    - [Other Creation Methods](#other-creation-methods)
- [Deleting Models](#deleting-models)
    - [Soft Deleting](#soft-deleting)
    - [Querying Soft Deleted Models](#querying-soft-deleted-models)
- [Query Scopes](#query-scopes)
    - [Global Scopes](#global-scopes)
    - [Local Scopes](#local-scopes)
- [Comparing Models](#comparing-models)
- [Events](#events)
    - [Observers](#observers)

<a name="introduction"></a>
## Introduction

The Eloquent ORM included with Laravel provides a beautiful, simple ActiveRecord implementation for working with your database. Each database table has a corresponding "Model" which is used to interact with that table. Models allow you to query for data in your tables, as well as insert new records into the table.

Before getting started, be sure to configure a database connection in `config/database.php`. For more information on configuring your database, check out [the documentation](/docs/database#configuration).

<a name="retrieving-models"></a>
## Retrieving Models

Once you have created a model and [its associated database table](/docs/migrations#writing-migrations), you are ready to start retrieving data from your database. Think of each Eloquent model as a powerful [query builder](/docs/queries) allowing you to fluently query the database table associated with the model. For example:

```try-code
{
    mode: "cli",
    cliCode: `
Schema::create('flights', function ($table) {
    $table->bigIncrements('id');
    $table->string('name');
    $table->string('number');
    $table->timestamps();
});

class Flight extends Illuminate\\Database\\Eloquent\\Model {
  
}

Flight::forceCreate([
    'name' => 'Flight 1',
    'number' => 'AB6650'
]);

$flights = Flight::all();

foreach ($flights as $flight) {
    echo $flight->name;
}
`
}
```

    <?php

    $flights = App\Flight::all();

    foreach ($flights as $flight) {
        echo $flight->name;
    }

#### Adding Additional Constraints

The Eloquent `all` method will return all of the results in the model's table. Since each Eloquent model serves as a [query builder](/docs/queries), you may also add constraints to queries, and then use the `get` method to retrieve the results:


```try-code
{
    mode: "cli",
    cliCode: `// Create our model database schema
Schema::create('flights', function ($table) {
    $table->bigIncrements('id');
    $table->string('name');
    $table->string('number');
    $table->boolean('active')->default(true);
    $table->timestamps();
});

class Flight extends Illuminate\\Database\\Eloquent\\Model {
  
}

// Insert dummy models
Flight::forceCreate([
    'name' => 'Inactive Flight',
    'number' => 'AB6650',
    'active' => false,
]);

Flight::forceCreate([
    'name' => 'Active Flight',
    'number' => 'AB6650',
    'active' => true,
]);

// Retrieve models
$flights = Flight::where('active', 1)
               ->orderBy('name', 'desc')
               ->take(10)
               ->get();
`
}
```

    $flights = App\Flight::where('active', 1)
                   ->orderBy('name', 'desc')
                   ->take(10)
                   ->get();


#### Refreshing Models

You can refresh models using the `fresh` and `refresh` methods. The `fresh` method will re-retrieve the model from the database. The existing model instance will not be affected:

    $flight = App\Flight::where('number', 'FR 900')->first();

    $freshFlight = $flight->fresh();

The `refresh` method will re-hydrate the existing model using fresh data from the database. In addition, all of its loaded relationships will be refreshed as well:


```try-code
{
    mode: "cli",
    cliCode: `// Create our model database schema
Schema::create('flights', function ($table) {
    $table->bigIncrements('id');
    $table->string('name');
    $table->string('number');
    $table->boolean('active')->default(true);
    $table->timestamps();
});

class Flight extends Illuminate\\Database\\Eloquent\\Model {
  
}

// Insert dummy models
Flight::forceCreate([
    'name' => 'Inactive Flight',
    'number' => 'FR 900',
]);

$flight = Flight::where('number', 'FR 900')->first();

$flight->number = 'FR 456';

$flight->refresh();

$flight->number;
`
}
```

    $flight = App\Flight::where('number', 'FR 900')->first();

    $flight->number = 'FR 456';

    $flight->refresh();

    $flight->number; // "FR 900"

<a name="collections"></a>
### Collections

For Eloquent methods like `all` and `get` which retrieve multiple results, an instance of `Illuminate\Database\Eloquent\Collection` will be returned. The `Collection` class provides [a variety of helpful methods](/docs/eloquent-collections#available-methods) for working with your Eloquent results:



```try-code
{
    mode: "cli",
    cliCode: `// Create our model database schema
Schema::create('flights', function ($table) {
    $table->bigIncrements('id');
    $table->string('name');
    $table->string('number');
    $table->boolean('cancelled')->default(false);
    $table->timestamps();
});

class Flight extends Illuminate\\Database\\Eloquent\\Model {
  
}

// Insert dummy models
Flight::forceCreate([
    'name' => 'Regular Flight',
    'number' => 'FR 900',
]);
Flight::forceCreate([
    'name' => 'Cancelled Flight',
    'number' => 'FR 900',
    'cancelled' => true,
]);

$flights = Flight::all();

$flights = $flights->reject(function ($flight) {
    return $flight->cancelled;
});
`
}
```

    $flights = $flights->reject(function ($flight) {
        return $flight->cancelled;
    });

You may also loop over the collection like an array:

    foreach ($flights as $flight) {
        echo $flight->name;
    }

<a name="chunking-results"></a>
### Chunking Results

If you need to process thousands of Eloquent records, use the `chunk` command. The `chunk` method will retrieve a "chunk" of Eloquent models, feeding them to a given `Closure` for processing. Using the `chunk` method will conserve memory when working with large result sets:

    Flight::chunk(200, function ($flights) {
        foreach ($flights as $flight) {
            //
        }
    });

The first argument passed to the method is the number of records you wish to receive per "chunk". The Closure passed as the second argument will be called for each chunk that is retrieved from the database. A database query will be executed to retrieve each chunk of records passed to the Closure.

#### Using Cursors

The `cursor` method allows you to iterate through your database records using a cursor, which will only execute a single query. When processing large amounts of data, the `cursor` method may be used to greatly reduce your memory usage:

    foreach (Flight::where('foo', 'bar')->cursor() as $flight) {
        //
    }

The `cursor` returns an `Illuminate\Support\LazyCollection` instance. [Lazy collections](/docs/collections#lazy-collections) allow you to use many of collection methods available on typical Laravel collections while only loading a single model into memory at a time:

    $users = App\User::cursor()->filter(function ($user) {
        return $user->id > 500;
    });

    foreach ($users as $user) {
        echo $user->id;
    }

<a name="advanced-subqueries"></a>
### Advanced Subqueries

#### Subquery Selects

Eloquent also offers advanced subquery support, which allows you to pull information from related tables in a single query. For example, let's imagine that we have a table of flight `destinations` and a table of `flights` to destinations. The `flights` table contains an `arrived_at` column which indicates when the flight arrived at the destination.

Using the subquery functionality available to the `select` and `addSelect` methods, we can select all of the `destinations` and the name of the flight that most recently arrived at that destination using a single query:


```try-code
{
    mode: "cli",
    cliCode: `// Create our model database schemas
Schema::create('flights', function ($table) {
    $table->bigIncrements('id');
    $table->integer('destination_id');
    $table->string('name');
    $table->string('number');
    $table->timestamps();
});

Schema::create('destinations', function ($table) {
    $table->bigIncrements('id');
    $table->string('name');
    $table->timestamps();
});

class Destination extends Illuminate\\Database\\Eloquent\\Model {

}

class Flight extends Illuminate\\Database\\Eloquent\\Model {

}

// Insert dummy models
Destination::forceCreate([
    'name' => 'Atlanta',
]);
Flight::forceCreate([
    'destination_id' => 1,
    'name' => 'Flight to Atlanta',
    'number' => 'FR 900',
]);

Destination::addSelect(['last_flight' => Flight::select('name')
    ->whereColumn('destination_id', 'destinations.id')
    ->orderBy('arrived_at', 'desc')
    ->limit(1)
])->get();

`
}
```

    use App\Destination;
    use App\Flight;

    return Destination::addSelect(['last_flight' => Flight::select('name')
        ->whereColumn('destination_id', 'destinations.id')
        ->orderBy('arrived_at', 'desc')
        ->limit(1)
    ])->get();

#### Subquery Ordering

In addition, the query builder's `orderBy` function supports subqueries. We may use this functionality to sort all destinations based on when the last flight arrived at that destination. Again, this may be done while executing a single query against the database:

    return Destination::orderByDesc(
        Flight::select('arrived_at')
            ->whereColumn('destination_id', 'destinations.id')
            ->orderBy('arrived_at', 'desc')
            ->limit(1)
    )->get();

<a name="retrieving-single-models"></a>
## Retrieving Single Models / Aggregates

In addition to retrieving all of the records for a given table, you may also retrieve single records using `find` or `first`. Instead of returning a collection of models, these methods return a single model instance:

```try-code
{
    mode: "cli",
    cliCode: `// Create our model database schemas
Schema::create('flights', function ($table) {
    $table->bigIncrements('id');
    $table->string('name');
    $table->string('number');
    $table->timestamps();
});

class Flight extends Illuminate\\Database\\Eloquent\\Model {

}

// Insert dummy models
Flight::forceCreate([
    'name' => 'Flight to Atlanta',
    'number' => 'FR 900',
]);

$flight = Flight::find(1);
`
}
```

    // Retrieve a model by its primary key...
    $flight = App\Flight::find(1);

    // Retrieve the first model matching the query constraints...
    $flight = App\Flight::where('active', 1)->first();

You may also call the `find` method with an array of primary keys, which will return a collection of the matching records:



```try-code
{
    mode: "cli",
    cliCode: `// Create our model database schemas
Schema::create('flights', function ($table) {
    $table->bigIncrements('id');
    $table->string('name');
    $table->string('number');
    $table->timestamps();
});

class Flight extends Illuminate\\Database\\Eloquent\\Model {

}

// Insert dummy models
Flight::forceCreate([
    'name' => 'Flight to Atlanta',
    'number' => 'FR 900',
]);
Flight::forceCreate([
    'name' => 'Flight to New York',
    'number' => 'AF 920',
]);

$flight = Flight::find([1, 2]);
`
}
```

    $flights = App\Flight::find([1, 2, 3]);

#### Not Found Exceptions

Sometimes you may wish to throw an exception if a model is not found. This is particularly useful in routes or controllers. The `findOrFail` and `firstOrFail` methods will retrieve the first result of the query; however, if no result is found, a `Illuminate\Database\Eloquent\ModelNotFoundException` will be thrown:


```try-code
{
    mode: "cli",
    cliCode: `// Create our model database schemas
Schema::create('flights', function ($table) {
    $table->bigIncrements('id');
    $table->string('name');
    $table->string('number');
    $table->timestamps();
});

class Flight extends Illuminate\\Database\\Eloquent\\Model {

}

$flight = Flight::findOrFail(1);
`
}
```

    $model = App\Flight::findOrFail(1);

    $model = App\Flight::where('legs', '>', 100)->firstOrFail();

If the exception is not caught, a `404` HTTP response is automatically sent back to the user. It is not necessary to write explicit checks to return `404` responses when using these methods:

    Route::get('/api/flights/{id}', function ($id) {
        return App\Flight::findOrFail($id);
    });

<a name="retrieving-aggregates"></a>
### Retrieving Aggregates

You may also use the `count`, `sum`, `max`, and other [aggregate methods](/docs/queries#aggregates) provided by the [query builder](/docs/queries). These methods return the appropriate scalar value instead of a full model instance:

```try-code
{
    mode: "cli",
    cliCode: `// Create our model database schemas
Schema::create('flights', function ($table) {
    $table->bigIncrements('id');
    $table->string('name');
    $table->string('number');
    $table->boolean('active')->default(true);
    $table->timestamps();
});

class Flight extends Illuminate\\Database\\Eloquent\\Model {

}

// Insert dummy models
Flight::forceCreate([
    'name' => 'Flight to Atlanta',
    'number' => 'FR 900',
    'active' => true,
]);

$count = Flight::where('active', 1)->count();
`
}
```

    $count = App\Flight::where('active', 1)->count();

    $max = App\Flight::where('active', 1)->max('price');

<a name="inserting-and-updating-models"></a>
## Inserting & Updating Models

<a name="inserts"></a>
### Inserts

To create a new record in the database, create a new model instance, set attributes on the model, then call the `save` method:



```try-code
{
    mode: "cli",
    cliCode: `// Create our model database schemas
Schema::create('flights', function ($table) {
    $table->bigIncrements('id');
    $table->string('name');
    $table->string('number');
    $table->boolean('active')->default(true);
    $table->timestamps();
});

class Flight extends Illuminate\\Database\\Eloquent\\Model {

}

$flight = new Flight();

$flight->name = "Flight Name";

$flight->save();
`
}
```

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use App\Flight;
    use Illuminate\Http\Request;

    class FlightController extends Controller
    {
        /**
         * Create a new flight instance.
         *
         * @param  Request  $request
         * @return Response
         */
        public function store(Request $request)
        {
            // Validate the request...

            $flight = new Flight;

            $flight->name = $request->name;

            $flight->save();
        }
    }

In this example, we assign the `name` parameter from the incoming HTTP request to the `name` attribute of the `App\Flight` model instance. When we call the `save` method, a record will be inserted into the database. The `created_at` and `updated_at` timestamps will automatically be set when the `save` method is called, so there is no need to set them manually.

<a name="updates"></a>
### Updates

The `save` method may also be used to update models that already exist in the database. To update a model, you should retrieve it, set any attributes you wish to update, and then call the `save` method. Again, the `updated_at` timestamp will automatically be updated, so there is no need to manually set its value:


```try-code
{
    mode: "cli",
    cliCode: `// Create our model database schemas
Schema::create('flights', function ($table) {
    $table->bigIncrements('id');
    $table->string('name');
    $table->string('number');
    $table->boolean('active')->default(true);
    $table->timestamps();
});

class Flight extends Illuminate\\Database\\Eloquent\\Model {

}

// Insert dummy models
Flight::forceCreate([
    'name' => 'Flight to Atlanta',
    'number' => 'FR 900',
    'active' => true,
]);

$flight = Flight::find(1);

$flight->name = 'New Flight Name';

$flight->save();

$flight;
`
}
```

    $flight = App\Flight::find(1);

    $flight->name = 'New Flight Name';

    $flight->save();

#### Mass Updates

Updates can also be performed against any number of models that match a given query. In this example, all flights that are `active` and have a `destination` of `San Diego` will be marked as delayed:

    App\Flight::where('active', 1)
              ->where('destination', 'San Diego')
              ->update(['delayed' => 1]);

The `update` method expects an array of column and value pairs representing the columns that should be updated.

> {note} When issuing a mass update via Eloquent, the `saving`, `saved`, `updating`, and `updated` model events will not be fired for the updated models. This is because the models are never actually retrieved when issuing a mass update.

<a name="mass-assignment"></a>
### Mass Assignment

You may also use the `create` method to save a new model in a single line. The inserted model instance will be returned to you from the method. However, before doing so, you will need to specify either a `fillable` or `guarded` attribute on the model, as all Eloquent models protect against mass-assignment by default.

A mass-assignment vulnerability occurs when a user passes an unexpected HTTP parameter through a request, and that parameter changes a column in your database you did not expect. For example, a malicious user might send an `is_admin` parameter through an HTTP request, which is then passed into your model's `create` method, allowing the user to escalate themselves to an administrator.

So, to get started, you should define which model attributes you want to make mass assignable. You may do this using the `$fillable` property on the model. For example, let's make the `name` attribute of our `Flight` model mass assignable:


```try-code
{
    mode: "cli",
    cliCode: `// Create our model database schemas
Schema::create('flights', function ($table) {
    $table->bigIncrements('id');
    $table->string('name');
    $table->boolean('active')->default(true);
    $table->timestamps();
});

class Flight extends Illuminate\\Database\\Eloquent\\Model 
{
    /**
     * The attributes that are mass assignable.
     *
     * @var array
     */
    protected $fillable = ['name'];
}

$flight = Flight::create(['name' => 'Flight 10']);
`
}
```

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class Flight extends Model
    {
        /**
         * The attributes that are mass assignable.
         *
         * @var array
         */
        protected $fillable = ['name'];
    }

Once we have made the attributes mass assignable, we can use the `create` method to insert a new record in the database. The `create` method returns the saved model instance:

    $flight = App\Flight::create(['name' => 'Flight 10']);

If you already have a model instance, you may use the `fill` method to populate it with an array of attributes:

    $flight->fill(['name' => 'Flight 22']);

#### Guarding Attributes

While `$fillable` serves as a "white list" of attributes that should be mass assignable, you may also choose to use `$guarded`. The `$guarded` property should contain an array of attributes that you do not want to be mass assignable. All other attributes not in the array will be mass assignable. So, `$guarded` functions like a "black list". Importantly, you should use either `$fillable` or `$guarded` - not both. In the example below, all attributes **except for `price`** will be mass assignable:

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class Flight extends Model
    {
        /**
         * The attributes that aren't mass assignable.
         *
         * @var array
         */
        protected $guarded = ['price'];
    }

If you would like to make all attributes mass assignable, you may define the `$guarded` property as an empty array:

    /**
     * The attributes that aren't mass assignable.
     *
     * @var array
     */
    protected $guarded = [];

<a name="other-creation-methods"></a>
### Other Creation Methods

#### `firstOrCreate`/ `firstOrNew`

There are two other methods you may use to create models by mass assigning attributes: `firstOrCreate` and `firstOrNew`. The `firstOrCreate` method will attempt to locate a database record using the given column / value pairs. If the model can not be found in the database, a record will be inserted with the attributes from the first parameter, along with those in the optional second parameter.

The `firstOrNew` method, like `firstOrCreate` will attempt to locate a record in the database matching the given attributes. However, if a model is not found, a new model instance will be returned. Note that the model returned by `firstOrNew` has not yet been persisted to the database. You will need to call `save` manually to persist it:


```try-code
{
    mode: "cli",
    cliCode: `// Create our model database schemas
Schema::create('flights', function ($table) {
    $table->bigIncrements('id');
    $table->string('name');
    $table->boolean('active')->default(true);
    $table->timestamps();
});

class Flight extends Illuminate\\Database\\Eloquent\\Model 
{
    protected $fillable = ['name'];
}

$flight = Flight::firstOrCreate(['name' => 'Flight 10']);

$flight->wasRecentlyCreated;
`
}
```

    // Retrieve flight by name, or create it if it doesn't exist...
    $flight = App\Flight::firstOrCreate(['name' => 'Flight 10']);

    // Retrieve flight by name, or create it with the name, delayed, and arrival_time attributes...
    $flight = App\Flight::firstOrCreate(
        ['name' => 'Flight 10'],
        ['delayed' => 1, 'arrival_time' => '11:30']
    );

    // Retrieve by name, or instantiate...
    $flight = App\Flight::firstOrNew(['name' => 'Flight 10']);

    // Retrieve by name, or instantiate with the name, delayed, and arrival_time attributes...
    $flight = App\Flight::firstOrNew(
        ['name' => 'Flight 10'],
        ['delayed' => 1, 'arrival_time' => '11:30']
    );

#### `updateOrCreate`

You may also come across situations where you want to update an existing model or create a new model if none exists. Laravel provides an `updateOrCreate` method to do this in one step. Like the `firstOrCreate` method, `updateOrCreate` persists the model, so there's no need to call `save()`:


    // If there's a flight from Oakland to San Diego, set the price to $99.
    // If no matching model exists, create one.
    $flight = App\Flight::updateOrCreate(
        ['departure' => 'Oakland', 'destination' => 'San Diego'],
        ['price' => 99, 'discounted' => 1]
    );

<a name="deleting-models"></a>
## Deleting Models

To delete a model, call the `delete` method on a model instance:

```try-code
{
    mode: "cli",
    cliCode: `// Create our model database schemas
Schema::create('flights', function ($table) {
    $table->bigIncrements('id');
    $table->string('name');
    $table->boolean('active')->default(true);
    $table->timestamps();
});

class Flight extends Illuminate\\Database\\Eloquent\\Model 
{
    protected $fillable = ['name'];
}

$flight = Flight::create(['name' => 'Flight 10']);

$flight = Flight::find(1);

$flight->delete();
`
}
```

    $flight = App\Flight::find(1);

    $flight->delete();

#### Deleting An Existing Model By Key

In the example above, we are retrieving the model from the database before calling the `delete` method. However, if you know the primary key of the model, you may delete the model without retrieving it by calling the `destroy` method.  In addition to a single primary key as its argument, the `destroy` method will accept multiple primary keys, an array of primary keys, or a [collection](/docs/collections) of primary keys:

    App\Flight::destroy(1);

    App\Flight::destroy(1, 2, 3);

    App\Flight::destroy([1, 2, 3]);

    App\Flight::destroy(collect([1, 2, 3]));

#### Deleting Models By Query

You can also run a delete statement on a set of models. In this example, we will delete all flights that are marked as inactive. Like mass updates, mass deletes will not fire any model events for the models that are deleted:

    $deletedRows = App\Flight::where('active', 0)->delete();

> {note} When executing a mass delete statement via Eloquent, the `deleting` and `deleted` model events will not be fired for the deleted models. This is because the models are never actually retrieved when executing the delete statement.

<a name="soft-deleting"></a>
### Soft Deleting

In addition to actually removing records from your database, Eloquent can also "soft delete" models. When models are soft deleted, they are not actually removed from your database. Instead, a `deleted_at` attribute is set on the model and inserted into the database. If a model has a non-null `deleted_at` value, the model has been soft deleted. To enable soft deletes for a model, use the `Illuminate\Database\Eloquent\SoftDeletes` trait on the model:


```try-code
{
    mode: "cli",
    cliCode: `// Create our model database schemas
Schema::create('flights', function ($table) {
    $table->bigIncrements('id');
    $table->string('name');
    $table->boolean('active')->default(true);
    $table->softDeletes();
    $table->timestamps();
});

class Flight extends Illuminate\\Database\\Eloquent\\Model 
{
    use Illuminate\\Database\\Eloquent\\SoftDeletes;

    protected $fillable = ['name'];
}

Flight::create(['name' => 'Flight 10']);

$flight = Flight::find(1);

$flight->delete();

$flight->trashed();
`
}
```

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;
    use Illuminate\Database\Eloquent\SoftDeletes;

    class Flight extends Model
    {
        use SoftDeletes;
    }

You should also add the `deleted_at` column to your database table. The Laravel [schema builder](/docs/migrations) contains a helper method to create this column:

    Schema::table('flights', function (Blueprint $table) {
        $table->softDeletes();
    });

Now, when you call the `delete` method on the model, the `deleted_at` column will be set to the current date and time. And, when querying a model that uses soft deletes, the soft deleted models will automatically be excluded from all query results.

To determine if a given model instance has been soft deleted, use the `trashed` method:

    if ($flight->trashed()) {
        //
    }

<a name="querying-soft-deleted-models"></a>
### Querying Soft Deleted Models

#### Including Soft Deleted Models

As noted above, soft deleted models will automatically be excluded from query results. However, you may force soft deleted models to appear in a result set using the `withTrashed` method on the query:


```try-code
{
    mode: "cli",
    cliCode: `// Create our model database schemas
Schema::create('flights', function ($table) {
    $table->bigIncrements('id');
    $table->string('name');
    $table->boolean('active')->default(true);
    $table->softDeletes();
    $table->timestamps();
});

class Flight extends Illuminate\\Database\\Eloquent\\Model 
{
    use Illuminate\\Database\\Eloquent\\SoftDeletes;

    protected $fillable = ['name'];
}

Flight::create(['name' => 'Flight 10']);
Flight::create(['name' => 'Not-Deleted Flight 10']);

$flight = Flight::find(1);

$flight->delete();

$flights = Flight::withTrashed()
                    ->where('active', 1)
                    ->get();
`
}
```

    $flights = App\Flight::withTrashed()
                    ->where('account_id', 1)
                    ->get();

The `withTrashed` method may also be used on a [relationship](/docs/eloquent-relationships) query:

    $flight->history()->withTrashed()->get();

#### Retrieving Only Soft Deleted Models

The `onlyTrashed` method will retrieve **only** soft deleted models:


```try-code
{
    mode: "cli",
    cliCode: `// Create our model database schemas
Schema::create('flights', function ($table) {
    $table->bigIncrements('id');
    $table->string('name');
    $table->boolean('active')->default(true);
    $table->softDeletes();
    $table->timestamps();
});

class Flight extends Illuminate\\Database\\Eloquent\\Model 
{
    use Illuminate\\Database\\Eloquent\\SoftDeletes;

    protected $fillable = ['name'];
}

Flight::create(['name' => 'Flight 10']);
Flight::create(['name' => 'Not Deleted Flight 10']);

$flight = Flight::find(1);

$flight->delete();

$flights = Flight::onlyTrashed()
                    ->where('active', 1)
                    ->get();
`
}
```

    $flights = App\Flight::onlyTrashed()
                    ->where('airline_id', 1)
                    ->get();

#### Restoring Soft Deleted Models

Sometimes you may wish to "un-delete" a soft deleted model. To restore a soft deleted model into an active state, use the `restore` method on a model instance:



```try-code
{
    mode: "cli",
    cliCode: `// Create our model database schemas
Schema::create('flights', function ($table) {
    $table->bigIncrements('id');
    $table->string('name');
    $table->boolean('active')->default(true);
    $table->softDeletes();
    $table->timestamps();
});

class Flight extends Illuminate\\Database\\Eloquent\\Model 
{
    use Illuminate\\Database\\Eloquent\\SoftDeletes;

    protected $fillable = ['name'];
}

Flight::create(['name' => 'Flight 10']);
Flight::create(['name' => 'Not Deleted Flight 10']);

$flight = Flight::find(1);

$flight->delete();

$flight->restore();

$flights = Flight::onlyTrashed()
                    ->get();
`
}
```

    $flight->restore();

You may also use the `restore` method in a query to quickly restore multiple models. Again, like other "mass" operations, this will not fire any model events for the models that are restored:

    App\Flight::withTrashed()
            ->where('airline_id', 1)
            ->restore();

Like the `withTrashed` method, the `restore` method may also be used on [relationships](/docs/eloquent-relationships):

    $flight->history()->restore();

#### Permanently Deleting Models

Sometimes you may need to truly remove a model from your database. To permanently remove a soft deleted model from the database, use the `forceDelete` method:



```try-code
{
    mode: "cli",
    cliCode: `// Create our model database schemas
Schema::create('flights', function ($table) {
    $table->bigIncrements('id');
    $table->string('name');
    $table->boolean('active')->default(true);
    $table->softDeletes();
    $table->timestamps();
});

class Flight extends Illuminate\\Database\\Eloquent\\Model 
{
    use Illuminate\\Database\\Eloquent\\SoftDeletes;

    protected $fillable = ['name'];
}

Flight::create(['name' => 'Flight 10']);

$flight = Flight::find(1);

$flight->forceDelete();
`
}
```

    // Force deleting a single model instance...
    $flight->forceDelete();

    // Force deleting all related models...
    $flight->history()->forceDelete();

<a name="query-scopes"></a>
## Query Scopes

<a name="global-scopes"></a>
### Global Scopes

Global scopes allow you to add constraints to all queries for a given model. Laravel's own [soft delete](#soft-deleting) functionality utilizes global scopes to only pull "non-deleted" models from the database. Writing your own global scopes can provide a convenient, easy way to make sure every query for a given model receives certain constraints.

#### Writing Global Scopes

Writing a global scope is simple. Define a class that implements the `Illuminate\Database\Eloquent\Scope` interface. This interface requires you to implement one method: `apply`. The `apply` method may add `where` constraints to the query as needed:

    <?php

    namespace App\Scopes;

    use Illuminate\Database\Eloquent\Builder;
    use Illuminate\Database\Eloquent\Model;
    use Illuminate\Database\Eloquent\Scope;

    class AgeScope implements Scope
    {
        /**
         * Apply the scope to a given Eloquent query builder.
         *
         * @param  \Illuminate\Database\Eloquent\Builder  $builder
         * @param  \Illuminate\Database\Eloquent\Model  $model
         * @return void
         */
        public function apply(Builder $builder, Model $model)
        {
            $builder->where('age', '>', 200);
        }
    }


#### Applying Global Scopes

To assign a global scope to a model, you should override a given model's `boot` method and use the `addGlobalScope` method:




```try-code
{
    mode: "cli",
    cliCode: `// Create our model database schemas
Schema::create('guests', function ($table) {
    $table->bigIncrements('id');
    $table->string('name');
    $table->integer('age');
    $table->timestamps();
});

class AgeScope implements Illuminate\\Database\\Eloquent\\Scope
{
    public function apply($builder, $model)
    {
        $builder->where('age', '>', 18);
    }
}

class Guest extends Illuminate\\Database\\Eloquent\\Model 
{
    protected $fillable = ['name', 'age'];

    protected static function boot()
    {
        parent::boot();

        static::addGlobalScope(new AgeScope);
    }
}

Guest::create(['name' => 'Paul', 'age' => 6]);
Guest::create(['name' => 'Marcel', 'age' => 34]);

Guest::all();
`
}
```

    <?php

    namespace App;

    use App\Scopes\AgeScope;
    use Illuminate\Database\Eloquent\Model;

    class User extends Model
    {
        /**
         * The "booting" method of the model.
         *
         * @return void
         */
        protected static function boot()
        {
            parent::boot();

            static::addGlobalScope(new AgeScope);
        }
    }

After adding the scope, a query to `User::all()` will produce the following SQL:

    select * from `users` where `age` > 200

#### Anonymous Global Scopes

Eloquent also allows you to define global scopes using Closures, which is particularly useful for simple scopes that do not warrant a separate class:



```try-code
{
    mode: "cli",
    cliCode: `// Create our model database schemas
Schema::create('guests', function ($table) {
    $table->bigIncrements('id');
    $table->string('name');
    $table->integer('age');
    $table->timestamps();
});

class Guest extends Illuminate\\Database\\Eloquent\\Model 
{
    protected $fillable = ['name', 'age'];

    protected static function boot()
    {
        parent::boot();

        static::addGlobalScope('age', function ($builder) {
            $builder->where('age', '>', 18);
        });
    }
}

Guest::create(['name' => 'Paul', 'age' => 6]);
Guest::create(['name' => 'Marcel', 'age' => 34]);

Guest::all();
`
}
```

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Builder;
    use Illuminate\Database\Eloquent\Model;

    class User extends Model
    {
        /**
         * The "booting" method of the model.
         *
         * @return void
         */
        protected static function boot()
        {
            parent::boot();

            static::addGlobalScope('age', function (Builder $builder) {
                $builder->where('age', '>', 200);
            });
        }
    }

#### Removing Global Scopes

If you would like to remove a global scope for a given query, you may use the `withoutGlobalScope` method. The method accepts the class name of the global scope as its only argument:

```try-code
{
    mode: "cli",
    cliCode: `// Create our model database schemas
Schema::create('guests', function ($table) {
    $table->bigIncrements('id');
    $table->string('name');
    $table->integer('age');
    $table->timestamps();
});

class Guest extends Illuminate\\Database\\Eloquent\\Model 
{
    protected $fillable = ['name', 'age'];

    protected static function boot()
    {
        parent::boot();

        static::addGlobalScope('age', function ($builder) {
            $builder->where('age', '>', 18);
        });
    }
}

Guest::create(['name' => 'Paul', 'age' => 6]);
Guest::create(['name' => 'Marcel', 'age' => 34]);

Guest::withoutGlobalScope('age')->get();
`
}
```


    User::withoutGlobalScope(AgeScope::class)->get();

Or, if you defined the global scope using a Closure:

    User::withoutGlobalScope('age')->get();

If you would like to remove several or even all of the global scopes, you may use the `withoutGlobalScopes` method:

    // Remove all of the global scopes...
    User::withoutGlobalScopes()->get();

    // Remove some of the global scopes...
    User::withoutGlobalScopes([
        FirstScope::class, SecondScope::class
    ])->get();

<a name="local-scopes"></a>
### Local Scopes

Local scopes allow you to define common sets of constraints that you may easily re-use throughout your application. For example, you may need to frequently retrieve all users that are considered "popular". To define a scope, prefix an Eloquent model method with `scope`.

Scopes should always return a query builder instance:

```try-code
{
    mode: "cli",
    cliCode: `// Create our model database schemas
Schema::create('guests', function ($table) {
    $table->bigIncrements('id');
    $table->string('name');
    $table->integer('votes');
    $table->boolean('active');
    $table->timestamps();
});

class Guest extends Illuminate\\Database\\Eloquent\\Model 
{
    protected $fillable = ['name', 'votes', 'active'];


    /**
     * Scope a query to only include popular users.
     *
     * @param  \Illuminate\Database\Eloquent\Builder  $query
     * @return \Illuminate\Database\Eloquent\Builder
     */
    public function scopePopular($query)
    {
        return $query->where('votes', '>', 100);
    }

    /**
     * Scope a query to only include active users.
     *
     * @param  \Illuminate\Database\Eloquent\Builder  $query
     * @return \Illuminate\Database\Eloquent\Builder
     */
    public function scopeActive($query)
    {
        return $query->where('active', 1);
    }
}

Guest::create(['name' => 'Paul', 'votes' => 5000, 'active' => true]);
Guest::create(['name' => 'Marcel', 'votes' => 25, 'active' => false]);

Guest::popular()->active()->orderBy('created_at')->get();
`
}
```

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class User extends Model
    {
        /**
         * Scope a query to only include popular users.
         *
         * @param  \Illuminate\Database\Eloquent\Builder  $query
         * @return \Illuminate\Database\Eloquent\Builder
         */
        public function scopePopular($query)
        {
            return $query->where('votes', '>', 100);
        }

        /**
         * Scope a query to only include active users.
         *
         * @param  \Illuminate\Database\Eloquent\Builder  $query
         * @return \Illuminate\Database\Eloquent\Builder
         */
        public function scopeActive($query)
        {
            return $query->where('active', 1);
        }
    }

#### Utilizing A Local Scope

Once the scope has been defined, you may call the scope methods when querying the model. However, you should not include the `scope` prefix when calling the method. You can even chain calls to various scopes, for example:

    $users = App\User::popular()->active()->orderBy('created_at')->get();

Combining multiple Eloquent model scopes via an `or` query operator may require the use of Closure callbacks:

    $users = App\User::popular()->orWhere(function (Builder $query) {
        $query->active();
    })->get();

However, since this can be cumbersome, Laravel provides a "higher order" `orWhere` method that allows you to fluently chain these scopes together without the use of Closures:

    $users = App\User::popular()->orWhere->active()->get();

#### Dynamic Scopes

Sometimes you may wish to define a scope that accepts parameters. To get started, just add your additional parameters to your scope. Scope parameters should be defined after the `$query` parameter:

```try-code
{
    mode: "cli",
    cliCode: `// Create our model database schemas
Schema::create('guests', function ($table) {
    $table->bigIncrements('id');
    $table->string('name');
    $table->string('type');
    $table->timestamps();
});

class Guest extends Illuminate\\Database\\Eloquent\\Model 
{
    protected $fillable = ['name', 'type'];

    /**
     * Scope a query to only include users of a given type.
     *
     * @param  \Illuminate\Database\Eloquent\Builder  $query
     * @param  mixed  $type
     * @return \Illuminate\Database\Eloquent\Builder
     */
    public function scopeOfType($query, $type)
    {
        return $query->where('type', $type);
    }
}

Guest::create(['name' => 'Paul', 'type' => 'admin']);
Guest::create(['name' => 'Marcel', 'type' => 'user']);

Guest::ofType('admin')->get();
`
}
```

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class User extends Model
    {
        /**
         * Scope a query to only include users of a given type.
         *
         * @param  \Illuminate\Database\Eloquent\Builder  $query
         * @param  mixed  $type
         * @return \Illuminate\Database\Eloquent\Builder
         */
        public function scopeOfType($query, $type)
        {
            return $query->where('type', $type);
        }
    }

Now, you may pass the parameters when calling the scope:

    $users = App\User::ofType('admin')->get();

<a name="comparing-models"></a>
## Comparing Models

Sometimes you may need to determine if two models are the "same". The `is` method may be used to quickly verify two models have same primary key, table, and database connection:

    if ($post->is($anotherPost)) {
        //
    }

<a name="events"></a>
## Events

Eloquent models fire several events, allowing you to hook into the following points in a model's lifecycle: `retrieved`, `creating`, `created`, `updating`, `updated`, `saving`, `saved`, `deleting`, `deleted`, `restoring`, `restored`. Events allow you to easily execute code each time a specific model class is saved or updated in the database. Each event receives the instance of the model through its constructor.

The `retrieved` event will fire when an existing model is retrieved from the database. When a new model is saved for the first time, the `creating` and `created` events will fire. If a model already existed in the database and the `save` method is called, the `updating` / `updated` events will fire. However, in both cases, the `saving` / `saved` events will fire.

> {note} When issuing a mass update or delete via Eloquent, the `saved`, `updated`, `deleting`, and `deleted` model events will not be fired for the affected models. This is because the models are never actually retrieved when issuing a mass update or delete.

To get started, define a `$dispatchesEvents` property on your Eloquent model that maps various points of the Eloquent model's lifecycle to your own [event classes](/docs/events):

    <?php

    namespace App;

    use App\Events\UserDeleted;
    use App\Events\UserSaved;
    use Illuminate\Foundation\Auth\User as Authenticatable;

    class User extends Authenticatable
    {
        use Notifiable;

        /**
         * The event map for the model.
         *
         * @var array
         */
        protected $dispatchesEvents = [
            'saved' => UserSaved::class,
            'deleted' => UserDeleted::class,
        ];
    }

After defining and mapping your Eloquent events, you may use [event listeners](https://laravel.com/docs/events#defining-listeners) to handle the events.

<a name="observers"></a>
### Observers

#### Defining Observers

If you are listening for many events on a given model, you may use observers to group all of your listeners into a single class. Observers classes have method names which reflect the Eloquent events you wish to listen for. Each of these methods receives the model as their only argument. The `make:observer` Artisan command is the easiest way to create a new observer class:

    php artisan make:observer UserObserver --model=User

This command will place the new observer in your `App/Observers` directory. If this directory does not exist, Artisan will create it for you. Your fresh observer will look like the following:

    <?php

    namespace App\Observers;

    use App\User;

    class UserObserver
    {
        /**
         * Handle the User "created" event.
         *
         * @param  \App\User  $user
         * @return void
         */
        public function created(User $user)
        {
            //
        }

        /**
         * Handle the User "updated" event.
         *
         * @param  \App\User  $user
         * @return void
         */
        public function updated(User $user)
        {
            //
        }

        /**
         * Handle the User "deleted" event.
         *
         * @param  \App\User  $user
         * @return void
         */
        public function deleted(User $user)
        {
            //
        }

        /**
         * Handle the User "forceDeleted" event.
         *
         * @param  \App\User  $user
         * @return void
         */
        public function forceDeleted(User $user)
        {
            //
        }
    }

To register an observer, use the `observe` method on the model you wish to observe. You may register observers in the `boot` method of one of your service providers. In this example, we'll register the observer in the `AppServiceProvider`:

    <?php

    namespace App\Providers;

    use App\Observers\UserObserver;
    use App\User;
    use Illuminate\Support\ServiceProvider;

    class AppServiceProvider extends ServiceProvider
    {
        /**
         * Register any application services.
         *
         * @return void
         */
        public function register()
        {
            //
        }

        /**
         * Bootstrap any application services.
         *
         * @return void
         */
        public function boot()
        {
            User::observe(UserObserver::class);
        }
    }
