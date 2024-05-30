# elytica-laravel-Tutorial Part 1
Diet problem with Laravel Filament and elytica

# Getting started
To make life easier install:
* On Windows --> [Herd](https://herd.laravel.com/windows)
* On MacOS --> [Herd](https://herd.laravel.com/)

This should configure `PHP`, `Nginx` and you should be able to use `composer` from the terminal.<br>
Open a new terminal or PowerShell, and `cd <path>` to the `<path>` you want your projects.
Now run:
```
herd park
```
You should be able to create a laravel project in that directory using:
```
composer create-project laravel/laravel diet-app
cd diet-app
```
Now run:
```
herd link diet-app
```
You should be able to see you application [diet-app.test](http://diet-app.test)

### Installing a default panel
Run:
```
php artisan filament:install --panels
```
Create a new panel with ID `diet-app`, you should be able to access the panel with [diet-app.test/diet-app](http://diet-app.test/diet-app)

### Disable the login and use elytica as login options
Let's disable the login of filament:<br>
Remove `->login()` in `app/Providers/Filament/DietAppPanelProvider.php`<br>
Install socialite and the elytica socialite package
```
composer require laravel/socialite
composer require elytica/elytica-socialite
php artisan vendor:publish --provider=Elytica\Socialite\ElyticaServiceProvider
php artisan migrate
```
In `resources/views/welcome.blade.php` you can add:
```
<a class="ms-3" href="{{ route('elytica_service.auth') }}">                         
    {{ __('Log in') }}                                                              
</a> 
```
and in your `.env`:
```
ELYTICA_SERVICE_CLIENT_ID=                                       
ELYTICA_SERVICE_CLIENT_SECRET=
ELYTICA_SERVICE_REDIRECT_URI=
```
To generate the following, you need to have a developer role with you elytica account, this can be achieved by sending an email to `info@elytica.com`.
Go to `Tokens->Create New Client->Create an OAuth Client`, and fill in the details.
The redirect details will be:
* http://diet-app.test/auth/callback
This will allow you to create a third party client.

After this, add the following routes in `routes/auth.php`:
```
Route::middleware('guest')->group(function () {
    Route::get('/auth/redirect', function () {
        return Socialite::driver('elytica_service')->redirect();
    })->name('elytica_service.auth');

    Route::get('/auth/callback', function () {
       $user = Socialite::driver('elytica_service')->user();
       $authUser = User::updateOrCreate(
           ['email' => $user->getEmail()],
           [
            'name' => $user->getName(),
            'elytica_service_id' => $user->getId(),
            'elytica_service_token' => $user->token,
            'elytica_service_expires_in' => $user->expiresIn,
            'elytica_service_refreshToken' => $user->refreshToken,
           ]
       );
       Auth::login($authUser, true);
       return redirect('/diet-app');
    });

    Route::get('login', function () {
        return view('welcome');
    })->name('login');
});

Route::middleware('auth')->group(function () {
    Route::post('logout', function(Request $request) {
        Auth::guard('web')->logout();
        $request->session()->invalidate();
        $request->session()->regenerateToken();
        return redirect('/');
    })->name('logout');
});
```

You should be able to login to your new application using elytica Auth!
