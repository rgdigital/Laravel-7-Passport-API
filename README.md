# Create an Oauth2 resource server (API) with Laravel 7 + Passport

## API

### Setting up Laravel with Passport

1. With Laravel installed, `cd` into your project directory and run `composer create-project --prefer-dist laravel/laravel ./`.

2. Install Passport with `composer require laravel/passport`.

3. Add your database credentials to your .env file (in Laravels root directory).

4. Create the database tables with `php artisan migrate`.

**Note:** If you get an error of `[PDOException] SQLSTATE[42000]: Syntax error or access violation: 1071 Specified key was too long; max key length is 767 bytes`, then you need to add this code to `app/providers/AppServiceProvider.php`:

```php
use Illuminate\Support\Facades\Schema;

public function boot()
{
    Schema::defaultStringLength(191);
}
```

5. After running this command, add the `Laravel\Passport\HasApiTokens` trait to your `App\User` model (in app/User.php). This trait will provide a few helper methods to your model which allow you to inspect the authenticated user's token and scopes:

```php
<?php

namespace App;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Passport\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, Notifiable;
}
```

6. Next, you should call the Passport::routes method within the boot method of your AuthServiceProvider. This method will register the routes necessary to issue access tokens and revoke access tokens, clients, and personal access tokens:

```php
<?php

namespace App\Providers;

use Illuminate\Foundation\Support\Providers\AuthServiceProvider as ServiceProvider;
use Illuminate\Support\Facades\Gate;
use Laravel\Passport\Passport;

class AuthServiceProvider extends ServiceProvider
{
    /**
     * The policy mappings for the application.
     *
     * @var array
     */
    protected $policies = [
        'App\Model' => 'App\Policies\ModelPolicy',
    ];

    /**
     * Register any authentication / authorization services.
     *
     * @return void
     */
    public function boot()
    {
        $this->registerPolicies();

        Passport::routes();
    }
}
```

7. Finally, in your config/auth.php configuration file, you should set the driver option of the api authentication guard to passport. This will instruct your application to use Passport's TokenGuard when authenticating incoming API requests:

```php
'guards' => [
    'web' => [
        'driver' => 'session',
        'provider' => 'users',
    ],

    'api' => [
        'driver' => 'passport',
        'provider' => 'users',
    ],
],
```

8. Now we need to add UI + Vue component capability to our laravel app with `composer require laravel/ui`, then `php artisan ui vue --auth`

9. Now we build the UI components with `npm run dev`

**Note**: If you get an error of `'cross-env' is not recognized as an internal or external command, operable program or batch file.` then you can use [this fix](https://stackoverflow.com/a/45629368/737794):
- Remove `node_modules` folder
- Run `npm install --global cross-env`
- Run `npm install --no-bin-links`
- Run `npm run dev`

### Adding a UI for Passport token management

1. To add login UI components to the app we use `php artisan vendor:publish --tag=passport-components`.


2. Now we need to add the Passport token Vue components to our UI. We have a login system sorted so you can go ahead and register via the register link in the header of your app. Once registered you will now be logged into your account in the app. 

3. Now we need to register the Passport Vue components in our `app.js`:

```javascript
Vue.component(
    'passport-clients',
    require('./components/passport/Clients.vue').default
);

Vue.component(
    'passport-authorized-clients',
    require('./components/passport/AuthorizedClients.vue').default
);

Vue.component(
    'passport-personal-access-tokens',
    require('./components/passport/PersonalAccessTokens.vue').default
);
```

4. Add the components to your JS build with `npm run dev`.

5. Now we can add the Passport token Vue components to the logged in home view (resources/views/home.blade.php) so it looks like this:

```php
@extends('layouts.app')

@section('content')
<div class="container">
    <div class="row justify-content-center">
        <div class="col-md-8">
            <passport-clients></passport-clients>
            <passport-authorized-clients></passport-authorized-clients>
            <passport-personal-access-tokens></passport-personal-access-tokens>
        </div>
    </div>
</div>
@endsection

```

6. Now run `php artisan serve` and go to [localhost:8000](http://localhost:8000) in your browser.

7. Login using the header nav links in the top-right, then you will be able to see our Vue components for token management.

## Client / Consumer Apps

Choosing a grant type depends on how you want your API to be accessed:

- **Password**: Get an access token by sending your username + password in the API request.

### 'Password' grant type

1. First we need to start a new app, so `cd` into your project directory and run `composer create-project --prefer-dist laravel/laravel ./`.

2. Add GuzzleHttp to make API requests with `composer require guzzlehttp/guzzle`.

3. Add this code in `app/User.php` to retrieve and access token:

```php
    $http = new GuzzleHttp\Client;

    try {
        $response = $http->post('http://api.pangolincomms.com/oauth/token', [
            'form_params' => [
                'grant_type' => 'password',
                'client_id' => '5',
                'client_secret' => '29b9gRB8rt8pJV5XQNPWet6FS3WnBRaY5j8MAjB8',
                'username' => 'rick@rgdigital.io',
                'password' => '1337Hacksaw',
                'scope' => '*',
            ],
        ]);

        return json_decode((string) $response->getBody(), true);
    } catch (GuzzleHttp\Exception\BadResponseException $e) {
        // This will output the error screen from your API - not the consumer app it is fired from
        echo $e->getResponse()->getBody()->getContents();
    }
```

4. Now if you try to access `yourconsumerapp.com/redirect` you should see a JSON response with an access token.

**Note**: If your API call is looking at the wrong database (eg, you get an error response for the consumer app DB instead of the API DB), you can run `php artisan config:cache` to clear your consumer apps cache.

5. Once that's working you can use that response to get the access token, then you can retrieve data from API endpoints. Here's the new code:

```php
    $http = new GuzzleHttp\Client;

    try {
        $response = $http->post('http://api.pangolincomms.com/oauth/token', [
            'form_params' => [
                'grant_type' => 'password',
                'client_id' => '5',
                'client_secret' => '29b9gRB8rt8pJV5XQNPWet6FS3WnBRaY5j8MAjB8',
                'username' => 'rick@rgdigital.io',
                'password' => '1337Hacksaw',
                'scope' => '*',
            ],
        ]);

        $auth = json_decode( (string) $response->getBody() );
        $response = $http->get('http://api.pangolincomms.com/api/users', [
            'headers' => [
                'Authorization' => 'Bearer '.$auth->access_token,
            ]
        ]);
        $users = json_decode( (string) $response->getBody());

        print_r($users);
        
    } catch (GuzzleHttp\Exception\BadResponseException $e) {
        echo $e->getResponse()->getBody()->getContents();
    }
```

Now when you refresh the `/redirect` page you will get an object with the user details of the user you specified in the API call.