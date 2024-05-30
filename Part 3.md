# elytica-laravel Tutorial Part 3 - Using the elytica Compute Client

We can start by installing the elytica compute client package:

```
rm composer.lock
composer require elytica/compute-client
```

We will also require a webhook client so [elytica](https://service.elytica.com) can come back with results
```
composer require spatie/laravel-webhook-client
php artisan vendor:publish --provider="Spatie\WebhookClient\WebhookClientServiceProvider" --tag="webhook-client-config"
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
