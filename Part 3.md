# elytica-laravel Tutorial Part 3 - Using the elytica Compute Client

We can start by installing the elytica compute client package:

```
rm composer.lock
composer require elytica/compute-client
```
## Creating a project with the elytica service
There are multiple ways to utilize the project/job structure of elytica. You can create a new project for each run, or your can create a single project with multiple jobs, or the easiest to get started is to create a single project and job and reuse them. In this tutorial we will create a project the first time the user is logged in and configure each run as a different job on in the elytica service.<br>
Before we begin, let’s discuss why using Elytica for computation is advantageous. Firstly, offloading computationally intensive and potentially long-running jobs from your web server helps prevent it from being overburdened. While Laravel jobs can handle this, they come with several challenges. Managing computational resources, memory, and CPU usage on a queue server can be difficult and time-consuming. Additionally, setting up and maintaining solvers and dependencies on your queue server can quickly become cumbersome. Using Elytica simplifies these tasks, allowing for efficient and effective computation management. You also get access to our modelling language :)<br>

Here is our approach - we will create the idea of an optimization run, this optimization run will use all the current available food and nutritional value, along with the constraints for that run and save the results.
You can see this "run" as an optimization run for a specific person, e.g.
|                  | Men                | Women              |
|------------------|--------------------|--------------------|
| Energy (kcal)    | 2,500 to 3,000     | 1,800 to 2,400     |
| Protein (g)      | 56                 | 46                 |
| Carbohydrates (g)| 281 to 406         | 225 to 325         |
| Fat (g)          | 70 to 105          | 55 to 80           |
| Saturates (g)    | Less than 30       | Less than 20       |
| Salt (g)         | Less than 2.3      | Less than 2.3      |

> **Note:** These values are based on general dietary guidelines and can vary based on individual health conditions, lifestyle factors, and nutritional requirements. For personalized dietary advice, consult a healthcare professional or dietitian.

In our case we want to create a table to hold individual health conditions / profiles, lets call the model `Profile` so the relating table will be `profiles`. This table will ha a foreign key to the `users` table, a `name` and an `elytica_job_id`.
You can see each `Profile` as an optimization run, given nutritional constraints assosiated with the profile.
Lets start by creating a model, migration and a filament resource as we did in the previous tutorials:
```
php artisan make:model Profile -m
```
Add the following fields to `database/migrations/xxxx_xx_xx_xxxxxx_create_profiles_table.php` and run `php artisan migrate` afterwards:
```
            $table->foreignIdFor(User::class);
            $table->unsignedBigInteger(column: 'elytica_job_id')->nullable();
            $table->string(column: 'name');
```
Create a filament resource:
```
php artisan make:filament-resource Profile --generate
```
Once again you can remove:
```
                Forms\Components\TextInput::make('user_id')
                    ->required()
                    ->numeric(),

```
from `form(...)` in `app/Filament/Resources/ProfileResource.php` and add the following in `app/Models/Profile.php`:
```
    protected $guarded = ['user_id'];
    protected static function boot()
    {
        parent::boot();
        static::creating(function ($profile) {
            $profile->user_id = auth()->id();
        });
        static::updating(function ($profile) {
            if ($profile->isDirty('user_id')) {
                $profile->user_id = auth()->id();
            }
        });
    }
```
There is no need for us to see `elytica_job_id` in the form or table view of filament, but for now we will leave it in to do debugging.
We can not yet run an elytica job, since there is no elytica project created, so lets start with creating a project and storing the project id. We need to add a `elytica_project_id` field to the user:
```
php artisan make:migration add_field_to_users_table
```
having the command in this form, automagically creates an empty migration for the `users` table, now we can add the field in `database/migrations/xxxx_xx_xx_xxxxxx_add_field_to_users_table.php` and run `php artisan migrate`:
```
    public function up(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->unsignedBigInteger(column: 'elytica_project_id')->nullable();
        });
    }
    public function down(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->dropColumn(columns: ['elytica_project_id']); /// <--- this is for php artisan migrate:rollback
        });
    }
```
We make the `elytica_project_id` nullable since we may not know if it exists. Now, we have a place to store or update our Elytica project (a project can have multiple jobs associated with it).

We can create a project once you sign up or, alternatively, whenever we optimize a profile. At this stage, the constraints are not associated with the profile yet, but we can add that later on. Let's first get a way to run an arbitrary job of a project.
At this stage, create a directory in `app/` called `Services`, and create a new class called `ElyticaService` (`app/Services/ElyticaService.php`):
```
<?php
namespace App\Services;

class ElyticaService
{

}
```
With php, we can always inject any service into the constructor of another class, and it will instantiate it for us.
We will move out logic to the service class, so we do not clutter our controllers or actions with the elytica compute logic. After this you will be able to reuse the alot of the ElyticaService code for other projects, an modify it according to your project needs.
Lets create a helper function in our service to initialize our compute server (`app/Services/ElyticaService.php`):
```
  use Elytica\ComputeClient\ComputeService;
```
and within the class:
```
public function initializeService(string $token): ?ComputeService { // <--- notice this type it is the same as null|ComputeService (php has union type support)
    if (!$token) {
        return null;
    }
    // handle token decryption here if needed
    // handle refresh token here before returning new ComputeService (not in this tutorial)
    return new ComputeService($token);
  }

```
Here we can use the token from `auth()->user()->elytica_service_token` - in the case you encrypted the token, you need to decrypt in before passing it to the `ComputeService` constructor.
To create a project add the following helper function:
```
    public function createOrUpdateProject(ComputeService $computeService, int|null $project_id): int|null {
        $applications = $computeService->getApplications();
        $appId = config('app.elytica_app_id');
        $appExists = array_filter($applications, fn($v) => $v->id == $appId);

        if (empty($appExists)) {
            return null;
        }

        $projects = $computeService->getProjects();
        if (!in_array($project_id, array_column($projects, 'id'))) {
            $project_id = $computeService->createNewProject(
                'Diet Problem',
                config('app.name'),
                $appId,
                config('app.url') . "/webhook",
                config('webhook-client.configs.0.signing_secret')
            )?->id;
        } else {
            $computeService->updateProject(
                $project_id,
                config('app.url') . "/webhook",
                config('webhook-client.configs.0.signing_secret')
            );
        }

        return $project_id;
    }
```

Initially, this function retrieves all applications that the Elytica user has access to. These applications include the MIP interpreter with Python or LUA, as well as various solvers such as HiGHS, SCIP, or CLP/CBC. Within our `config/app.php` configuration file, we can specify our preferred application and verify whether the user has access to it, particularly for commercial solvers or custom applications:
```
    /*
    |--------------------------------------------------------------------------
    | elytica Application
    |--------------------------------------------------------------------------
    | This is the elytica application ID, it is used to identify a
    | compute application, such as MIP Interpreter with Python and HiGHS.
    | For your usecase, make sure use select the correct application, e.g.
    | Simulation is ID 31, and MIP Interpreter with Python and HiGHS is 14.
    |
    */
    'elytica_app_id' => env('ELYTICA_APP_ID', 14),

```
Let's test our `createOrUpdateProject` function as an action in `app/Filament/Actions` directory (create one if the directory does not exist), add the following action:
```
<?php
namespace App\Filament\Actions;

use App\Services\ElyticaService;
use Filament\Tables\Actions\Action;
use Filament\Actions\Concerns\CanCustomizeProcess;
use Filament\Notifications\Notification;
use Filament\Tables\Table;
use Illuminate\Database\Eloquent\Model;
use App\Models\User;

class QueueAction extends Action
{
    use CanCustomizeProcess;

    public static function getDefaultName(): ?string
    {
        return 'queue';
    }

    protected function setUp(): void
    {
        parent::setUp();
        $this->icon('heroicon-m-play');
        $this->color('gray');
        $this->action(function (): void {
            $this->process(function (array $data, Model $record, Table $table) {
            $service = new ElyticaService();
            $compute = $service->initializeService(
              auth()->user()->elytica_service_token
            );
            $project_id = $service->createOrUpdateProject($compute, auth()->user()->elytica_project_id);
            User::find(auth()->id())->update(['elytica_project_id' => $project_id]);
            Notification::make()
                ->title("Queuing $record->name")
                ->success()
                ->send();
            });
        });
    }
}
```
In your filament profile resource - `app/Filament/Resources/ProfileResource.php` within your `table(...)` function, add your action to `$table`:
```
            ->actions([
                Tables\Actions\EditAction::make(),
                QueueAction::make(),
            ])

```
Notice the elegance of the builder pattern with a fluent interface used in the `table(...)` and `form(...)` functions' returns more about the builder pattern here. When you press queue on the profile, it should create a project on the Elytica service (no jobs or files are uploaded yet). Next, we want to create a job, and perhaps upload a file containing the model, followed by the data.
## Creating a job with the elytica service
In our `ElyticaService` class, we can add helper functions: one for creating a job, one for uploading the data and model, and later another for retrieving the results. As you can imagine, the model is available to the Elytica user since the computation is run from their account. If you want to hide the model, you should approach project and job creation differently. For instance, you could use your own account and pass the token from your .env file. This method contains the computation within your account but does so at the expense of your account's computational resources.

Let's create a `createJob` function for our service:
```
  public function createJob(ComputeService $computeService, int $project_id, Profile $profile)
  {
      $jobs = array_filter($computeService->getJobs($project_id), fn ($v) => $v->id == $profile->elytica_job_id);
      if (count($jobs) == 0) {
          return $computeService->createNewJob($project_id, "$profile->id");
      }
      return reset($jobs); // <--- this is a more elegant way than $jobs[0], reset() rewinds array's internal pointer to the first element and returns the value of the first array element.
  }
```
Now we can call the `createJob` function in out `QueueAction` class within out action function (`app/Filament/Actions/QueueAction.php`):
```
            $job = $service->createJob($compute, $project_id, $record);
            $record->elytica_job_id = $job->id;
            $record->save();
```
When you press queue now, there should be a job created for the project, you can view the project and job from [here](https://compute.elytica.com). There is no script assigned to the job, so let's create a new file that will be uploaded for that job, create a new file in `app/Services/file.hlpl` with the following content:
```
import json
f = open('data.json')
data = json.load(f)
print(data)
def main():
  print("Hello World")
  return  0
```
and create functions to upload the data and job script:
```
    public function collectData($job): Collection
    {
        return collect([]); // we will collect the correct data here later
    }
    public function uploadData(ComputeService|null $computeService, int|null $project_id, $job): array
    {
        if ($project_id === null || $computeService === null) {
            return [false, "Elytica project not used."];
        }

        $data = $computeService->uploadInputFile(
            "data.json",
            json_encode($this->collectData($job)),
            $project_id
        )?->newfiles;

        $script = $computeService->uploadInputFile(
            "$job->id.hlpl",
            file_get_contents(base_path('app/Services/file.hlpl')),
            $project_id
        )?->newfiles;

        if (!($data && $script)) {
            return [false, "Failed to upload files."];
        }

        $computeService->assignFileToJob($project_id, $job->id, $script[0]->id, 1);
        $computeService->assignFileToJob($project_id, $job->id, $data[0]->id, 2);

        return [true, "Success. You can now queue the job."];
    }
```
In your `QueueAction` class, you can add:
```
            $service->uploadData($compute, $project_id, $job);
```
Re-queue the job, and have a look on the [Interpreter](https://compute.elytica.com) if your file is there.
Finally we can add the `queueJob` function to the `ElyticaService` class to execute the script on Elytica.
```
  public function queueJob(ComputeService $computeService, int $project_id, $job) : array {
    [$ok, $err] = $this->uploadData($computeService, $project_id, $job);
    if ($ok) {
      $computeService->queueJob($job->id);
      return [$ok, ''];
    }
    return [false, $err];
  }
```
and replace 
```
$service->uploadData($compute, $project_id, $job);
```
with 
```
$service->queueJob($compute, $project_id, $job);
```
since our `queueJob` function uploads the data.

## Creating the profile constraints
This is the final data piece we need, creating the minimum and maximum values of nutrition for each profile. Create a model that holds the foreign keys for a `Profile` and a `Nutrition` along with a minimum and maximum.
```
php artisan make:model NutritionProfile -m 
```
and add the following fields to `database/migrations/xxxx_xx_xx_xxxxxx_create_nutrition_profiles_table.php`:
```
            $table->foreignIdFor(Nutrition::class);
            $table->foreignIdFor(Profile::class);
            $table->decimal(column: 'minimum', total: 20, places: 6);
            $table->decimal(column: 'maximum', total: 20, places: 6);
```
and run `php artisan migrate`.
Within the `NutritionProfile` model (`app/Models/NutritionProfile`) add the relations and user check:
```
    public function nutrition(): BelongsTo
    {
        return $this->belongsTo(Nutrition::class);
    }

    public function profile(): BelongsTo
    {
        return $this->belongsTo(Profile::class);
    }

    protected static function boot()
    {
        parent::boot();
        static::creating(function (NutritionProfile $nutritionProfile) {
            $profile = Food::find($nutritionProfile->profile_id);
            if ($profile->user_id !== auth()->id()) {
                throw new HttpException(403, 'You do not have permission to assign this profile.');
            }
        });
        static::updating(function (NutritionProfile $nutritionProfile) {
            $profile = Food::find($nutritionProfile->profile_id);
            if ($profile->user_id !== auth()->id()) {
                throw new HttpException(403, 'You do not have permission to assign this profile.');
            }
        });
    }
```
Once again we can generate the filament resource using:
```
php artisan make:filament-resource NutritionProfile --generate
```
We need to filter the profiles, such that a user can only see their own profiles. In the `form(...)` function of `app/Filament/Resources/NutritionProfileResource.php` we need to change the profile relation to filter on the auth user:
```
               Forms\Components\Select::make('profile_id')
                    ->relationship('profile', 'name', function (Builder $query) {
                        return $query->where('user_id', auth()->id());
                    })
```
and in the `table(...)`:
```
...
        return $table->modifyQueryUsing(fn (Builder $query) =>
               $query->whereIn('profile_id', function ($subquery) {
                   $subquery->select('id')
                            ->from('profiles')
                            ->where('user_id', auth()->id());
               })
            )->columns([
...
```
We should be ready to collect the data in the `ElyticaService` class (`app/Services/ElyticaService.php`):
```
    public function collectData($job): Collection
    {
        $profile = Profile::whereElyticaJobId($job?->id)->first();
        $food = Food::whereUserId($profile->user_id);
        return collect([
          'nutrition_profiles' => NutritionProfile::whereProfileId($profile?->id)->get(),
          'nutrition' => Nutrition::all(),
          'food' => $food->get(),
          'food_nutrition' => FoodNutrition::whereIn('food_id', $food->pluck('id'))->get()
        ]);
    }
```
We can update our script to solve the diet problem:
```
model diet_problem
  set N = load_nutrition()
  set F = load_food()
  const U = load_nutrition_max(), forall i in N
  const L = load_nutrition_min(), forall i in N
  const c = load_food_cost(), forall j in F
  const v = load_food_nutrients(), forall i in N, forall j in F
  var 0 <= x <= infty, forall j in F
  min sum_{j in F}{ c_{j}*x_{j} }
  constr sum_{j in F}{ v_{i,j}*x_{j} } <= U_{i}, forall i in N
  constr sum_{j in F}{ v_{i,j}*x_{j} } >= L_{i}, forall i in N
end

import elytica
import json

with open('data.json', 'r') as f:
  data = json.load(f)

load_food = lambda: [i['id'] for i in data["food"]]
load_nutrition = lambda: [i['id'] for i in data["nutrition"]]
load_nutrition_max = lambda: {i['nutrition_id']:i['maximum'] for i in data["nutrition_profiles"]}
load_nutrition_min = lambda: {i['nutrition_id']:i['minimum'] for i in data["nutrition_profiles"]}
load_food_cost = lambda: {i['id']:i['unit_cost'] for i in data["food"]}

load_food_nutrients = lambda: {f['id']:{ n['nutrition_id']: n['value'] for n in data['food_nutrition'] if n['food_id'] == f['id']} for f in data["food"]}
def main():
  #print(load_food())
  #print(load_nutrition())
  #print(load_nutrition_max())
  #print(load_nutrition_min())
  #print(load_food_cost())
  #print(load_food_nutrients())
  elytica.init_model("diet_problem")
  status = elytica.run_model("diet_problem")      
  if status == "FEASIBLE" or status == "OPTIMAL":
    F = elytica.get_model_set("diet_problem", "F")    
    for j in F:
      val = elytica.get_variable_value("diet_problem", f"x{j}")
      if val > 0 :
        row = {}
        row["food_id"] = j
        row["value"] = val
        solution.append(row)                   
    result={'solution': solution}
    elytica.write_results(json.dumps(result)) 
  return 0
```
More details on the diet problem can be found [here](https://blog.elytica.com/2020/11/10/tutorial-9-working-with-json-file-formats-diet-problem/).
