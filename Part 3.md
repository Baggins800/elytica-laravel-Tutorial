# elytica-laravel Tutorial Part 3 - Using the elytica Compute Client

We can start by installing the elytica compute client package:

```
rm composer.lock
composer require elytica/compute-client
```
## Listening for webhooks
We will also require a webhook client so [elytica](https://service.elytica.com) can come back with results
```
composer require spatie/laravel-webhook-client
php artisan vendor:publish --provider="Spatie\WebhookClient\WebhookClientServiceProvider" --tag="webhook-client-config"
```
The webhook route should also be added to `routes/web.php`:
```
Route::webhooks('webhook');
```
and we need a job to process the incomming webhooks from elytica, in `config/webhook-client.php` as:
```
            'process_webhook_job' => \App\Jobs\ProcessElyticaWebhook::class,
```
and create the job with:
```
php artisan make:job ProcessElyticaWebhook
```
in `app/Jobs/ProcessElyticaWebhook.php` we can start with the logic when a job is completed:
```
      $webhookData = $this->webhookCall->payload;
      $status = match($webhookData['status']) {
        0 => "NONE",
        1 => "QUEUED",
        2 => "ACCEPTED",
        3 => "PROCESSING",
        4 => "COMPLETED",
        5 => "STOPPED",
        default => "NONE"
      };
      // do something with if $status is COMPLETED
```
Make sure that the webhook job is of type `Spatie\WebhookClient\Jobs\ProcessWebhookJob` in `app/Jobs/ProcessElyticaWebhook.php`:
```
use Filament\Notifications\Notification; // <--- see below
use Spatie\WebhookClient\Jobs\ProcessWebhookJob;

class ProcessElyticaWebhook extends ProcessWebhookJob
{
...
```
You can also add a notification for the user when the webhooks comes in - see [Filament Notifications](https://filamentphp.com/docs/3.x/notifications/installation).
With Laravel you can configure the way your queue should be handled, a queue has the ability to listen for jobs, and using `php artisan queue:work`, the jobs can be executed.
Options for the queue includes `database`, `sync`, `redis`, `beanstalk` and amazon `sqs`. For development is it fine to use `sync` or `database`, but in production it is recommended to use something faster that is in memory, such as `redis` or `beanstalk` - you can read more on this topic in [this](https://medium.com/@noor1yasser9/best-practices-and-strategies-for-using-queues-in-laravel-7c3035f93e84) post.

## Creating a project with elytica
There are multiple ways to utilize the project/job structure of elytica. You can create a new project for each run, or your can create a single project with multiple jobs, or the easiest to get started is to create a single project and job and reuse them. In this tutorial we will create a project the first time the user is logged in and configure each run as a different job on in the elytica service.<br>
Before we begin, let’s discuss why using Elytica for computation is advantageous. Firstly, offloading computationally intensive and potentially long-running jobs from your web server helps prevent it from being overburdened. While Laravel jobs can handle this, they come with several challenges. Managing computational resources, memory, and CPU usage on a queue server can be difficult and time-consuming. Additionally, setting up and maintaining solvers and dependencies on your queue server can quickly become cumbersome. Using Elytica simplifies these tasks, allowing for efficient and effective computation management. You also get access to our modelling language :)<br>

Here is our approach - we will create the idea of an optimization run, this optimization run will use all the current available food and nutritional value, along with the constraints for that run and save the results.
You can see this "run" as an optimization run for a specific person, e.g.
| |Men|Women|
|-|---|-----|
|Energy (kcal)|2500 to 3200|2000 to 2500
|Protein (g)|45 to 55|45 to 55
|Carbohydrates (g)|250 to 300|230 to 270
|Fat (g)|90 to 100|65 to 75
|Saturates (g)|25 to 35|15 to 25
|Salt (g)|3 to 8|3 to 8

This table is just an illustration, please do not use these values as factual.

