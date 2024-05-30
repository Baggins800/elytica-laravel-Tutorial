# elytica-laravel-Tutorial Part 2

We will be using laravel filament to generate a new panel and create models.

## Creating a model to store nutrition
Let's make the nutritions system defined, and start by creating a Nutrition model:
```
php artisan make:model Nutrition -m
```
We will need an `id`, `name` and `unit` - edit the migration in 
```
database/migrations/xxxx_xx_xx_yyyyyyyy_create_nutrition_table.php
```
and add the following fields:
```
    $table->string('name')->unique();
    $table->string('unit');  
```

Lets populate the nutritions with some data using a seeder,
```
php artisan make:seeder NutritionSeeder
```
In `database/seeders/NutritionSeeder.php` we can add some seed values for our system:
```
        DB::table('nutrition')->insert([
            ['name' => 'Calories', 'unit' => 'kcal'],
            ['name' => 'Protein', 'unit' => 'g'],
            ['name' => 'Fat', 'unit' => 'g'],
            ['name' => 'Carbohydrates', 'unit' => 'g'],
            ['name' => 'Fiber', 'unit' => 'g'],
        ]);
```
And in `database/seeders/DatabaseSeeder.php`'s `run()` we can call our seeder with:
```
$this->call([
    NutritionSeeder::class,                                                                       
]); 
```
Now we can run `php artisan db:seed` and our database should be populated :)
Lets move on the the food.

## Creating a model to store food
```
php artisan make:model -m Food
```
This will create a migration for the `food` table along with the object relational model that accomodates it.

Each user should be able to define their own list of food; and there should be cost, name and the user associated with each food:
```
    $table->string('name');
    $table->double('unit_cost');
    $table->foreignIdFor(model: User::class);
```
As a user we need a place to define our food, here we can create a filament resource:

```
php artisan make:filament-resource FoodResource --generate
```

If you open up the page and add a food, you will get a 500 error, and you will see the obvious problem, users can add any id. To fix this we can remove:
```
Forms\Components\TextInput::make('user_id')
    ->required()                                                                      
    ->numeric(),
```
from the `form()` function in `app/Filament/Resources/FoodResource.php`. But now there is no way of knowing who the user is that is storing the food? Well, we can change the Food model in `app\Models\Food.php` to:
```
    protected $guarded = ['user_id'];
    protected static function boot()
    {
        parent::boot();
        static::creating(function ($food) {
            $food->user_id = auth()->id();
        });
        static::updating(function ($food) {
            if ($food->isDirty('user_id')) {
                $food->user_id = auth()->id();
            }
        });
    }
```
This will make our food secure. There is still another problem, our food can cost minus values, we can add `->minValue(0.0)` to:
```
                  Forms\Components\TextInput::make('unit_cost')                                             
                     ->required()
                     ->minValue(0.0) // < ----------------------------- add this here
                     ->numeric(),

```
We can now create and edit food!
