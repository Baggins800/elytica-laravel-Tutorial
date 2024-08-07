# elytica-laravel Tutorial Part 2 - Multi-Tenant Data Entry

We will be using Laravel Filament to generate a new panel and create models.

## Creating a model to store nutrition
Let's define the nutrition system, starting by creating a Nutrition model:

```
php artisan make:model Nutrition -m
```
We will need an id, name, and unit. Edit the migration in:

```
database/migrations/xxxx_xx_xx_yyyyyyyy_create_nutrition_table.php
```
and add the following fields:
```
    $table->string('name')->unique();
    $table->string('unit');  
```

You can run 
```
php artisan migrate
```
to create the table and 
```
php artisan db:table nutrition
```
to see the table structure.
Let's populate the nutrition data using a seeder:

```
php artisan make:seeder NutritionSeeder
```
In `database/seeders/NutritionSeeder.php`, we can add some seed values for our system:
```
        
        Illuminate\Support\Facades\DB::table('nutrition')->insert([
            ['name' => 'Calories', 'unit' => 'kcal'],
            ['name' => 'Protein', 'unit' => 'g'],
            ['name' => 'Fat', 'unit' => 'g'],
            ['name' => 'Carbohydrates', 'unit' => 'g'],
            ['name' => 'Fiber', 'unit' => 'g'],
        ]);
```
And in `database/seeders/DatabaseSeeder.php`, call our seeder in the `run()` method:

```
$this->call([
    NutritionSeeder::class,                                                                       
]); 
```
Now we can run php artisan db:seed and our database should be populated :)

Let's move on to the food.

## Creating a model to store food
```
php artisan make:model -m Food
```
This will create a migration for the food table along with the object-relational model that accommodates it.
The migration is located at `database/migrations/xxxx_xx_xx_yyyyyyyy_create_food_table.php`

Each user should be able to define their own list of food, including cost, name, and the user associated with each food:
```
    $table->string('name');
    $table->double('unit_cost');
    $table->foreignIdFor(model: App\Models\User::class);
```
After adding fields to the migration, remember to run `php artisan migrate` to create the table and fields.
In a production system with active users, you should never change historically created migrations but adding fields, you should instead create a new migration with `php artisan make:migration add_fields_to_sometablename_table`.
As a user, we need a place to define our food. Here we can create a Filament resource:

```
php artisan make:filament-resource FoodResource --generate
```
For brevity in this tutorial, we will disable Laravel's mass assignment protection. Filament only saves valid data to models so the models can be unguarded safely. To unguard all Laravel models at once, add `Model::unguard()` to the `boot()` method of `app/Providers/AppServiceProvider.php`:
```
use Illuminate\Database\Eloquent\Model;
 
public function boot(): void
{
    Model::unguard();
}
```
If you open up the page and add a food, you might get a 500 error because users can add any id. To fix this, remove:
```
Forms\Components\TextInput::make('user_id')
    ->required()                                                                      
    ->numeric(),
```
from the `form()` function in `app/Filament/Resources/FoodResource.php`. But now, there is no way of knowing who the user is that is storing the food. To fix this, change the Food model in `app/Models/Food.php` to:
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
This will make our food secure. There is still another problem - our food can have negative costs. We can add `->minValue(0.0)` to:
```
                  Forms\Components\TextInput::make('unit_cost')                                             
                     ->required()
                     ->minValue(0.0) // < ----------------------------- add this here
                     ->numeric(),

```
We can now create and edit food!<br>
When we go to the List view for Food, we see that the User id is visible and you can see all the users' data, which should not happen. We can modify the query beforehand.

In the table() function of app/Filament/Resources/FoodResource.php, modify the query and remove the user_id field using:
```
        return $table->modifyQueryUsing(fn (Builder $query) =>
               $query->where('user_id', auth()->id())
            )->columns([
                Tables\Columns\TextColumn::make('name')
                    ->searchable(),
                Tables\Columns\TextColumn::make('unit_cost')
                    ->numeric()
                    ->sortable(),
                Tables\Columns\TextColumn::make('created_at')
                    ->dateTime()
                    ->sortable()
                    ->toggleable(isToggledHiddenByDefault: true),
                Tables\Columns\TextColumn::make('updated_at')
                    ->dateTime()
                    ->sortable()
                    ->toggleable(isToggledHiddenByDefault: true),
            ])
```
## Adding nutrition data to food
Here we need to create a table that holds the data for our food and the system nutrition combination. We will start to see the power of Filament's creation form. Create a model:
```
php artisan make:model FoodNutrition -m
```
with the fields in the migration:

```
database/migrations/xxxx_xx_xx_yyyyyyyy_create_food_nutrition_table.php
```
as
```
            $table->foreignIdFor(App\Models\Food::class);                            
            $table->foreignIdFor(App\Models\Nutrition::class);
            $table->double(column: 'value');     
```
and run the migration `php artisan migrate`.
We can now define the relationships and assignable attributes in the `FoodNutrition` model in `app/Models/FoodNutrition.php`:
```
    protected $fillable = ['food_id', 'nutrition_id', 'value'];

    public function food() : BelongsTo
    {
        return $this->belongsTo(Food::class);
    }

    public function nutrition() : BelongsTo
    {
        return $this->belongsTo(Nutrition::class);
    }
```
Remember to include `use Illuminate\Database\Eloquent\Relations\BelongsTo;`. When you add the belongs to relation ships to the model, filament will be able to figure out that it should use the foreign keys to lookup the names of the related food and nutrition to display in a dropdown when creating or editing; this will become apparent once you generate the resources and attempt to add some data.

We can add this model as a Filament resource as well:
```
php artisan make:filament-resource FoodNutrition --generate
```
Filament generate will use the migrated table's fields to generate a form schema for the resource.
As with the `Food` model, when a user creates a `FoodNutrition`, they should only be able to create one for the food they own. In the FoodNutrition model (`app/Models/FoodNutrition.php`), add:

```
    protected static function boot()
    {
        parent::boot();
        static::creating(function (FoodNutrition $foodNutrition) {
            $food = Food::find($foodNutrition->food_id);
            if ($food->user_id !== auth()->id()) {
                throw new HttpException(403, 'You do not have permission to assign this food.');
            }
        });
        static::updating(function (FoodNutrition $foodNutrition) {
            $food = Food::find($foodNutrition->food_id);
            if ($food->user_id !== auth()->id()) {
                throw new HttpException(403, 'You do not have permission to assign this food.');
            }
        });
    }
```

We can also filter the options from the `form()` function in `app/Filament/Resources/FoodNutritionResource.php` using:
```
                    ->relationship('food', 'name', function (Builder $query) {
                        return $query->where('user_id', auth()->id());
                    })
```
* Note: `->relationship('food'...` - food refers to the FoodNutrition model's `food()` belongs to function.
This should work, but we want to see the unit of the selected nutrition. We can add a derived field in the Nutrition (`app/Models/Nutrition.php`) model using:

```
     protected $appends = ['name_and_unit'];
     public function getNameAndUnitAttribute() : string
     {
       return "$this->name ($this->unit)";
     }
```
and in our `FoodNutritionResource` change the `form()` function to have:
```
                 Forms\Components\Select::make('nutrition_id')
                     ->options(Nutrition::all()->pluck('name_and_unit', 'id'))
                     ->label('Nutrition')
                     ->required(),
```
