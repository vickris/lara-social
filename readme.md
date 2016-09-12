Laravel ships with Authentication and all you have to do is run `php artisan make:auth` and there you have it. From the views, controllers to the routes. And that is a great thing. However, this command will only make you a traditional login. Most sites nowadays when signing up, users have the option of signing up with a social provider such as Facebook. In this tutorial, I will teach you how to add multiple social providers to a Laravel app using  [Socialite package](https://github.com/laravel/socialite). For this tutorial we will add Facebook, Github and Twitter signup. The complete code can be found on [Github](https://github.com/andela-cvundi/lara-social)

Lets generate a new Laravel app. I am using Laravel 5.2 for this tutorial.
```bash
composer create-project laravel/laravel lara-social
```

### Migrations
If you look inside *database/migrations* you will notice that the generated laravel app already came with migrations for creating the *users table* and *password resets table*. Open the migration for creating the *users table*. There are a few things I want us to tweak before creating the *users* table.  A user created via OAuth will have a provider and we also need a provider Id which should be unique. With twitter, there are cases where the user does not have an email set up and thus the hash sent back by the callback won't have an email resulting to an error. To counter this, we will set the email field to nullable. Your `up method` in your migration should now look like this.

*database/migrations/{titmestamp}_create_users_table*
```php
[...]
public function up()
    {
        Schema::create('users', function (Blueprint $table) {
            $table->increments('id');
            $table->string('name');
            $table->string('email')->nullable();
            $table->string('password', 60);
            $table->string('provider');
            $table->string('provider_id')->unique();
            $table->rememberToken();
            $table->timestamps();
        });
    }
[...]
```

If you already have a Laravel app you will have to generate a new migration to add the extra columns to the *users table* and also set the email field to  `nullable` at the database level.

Then run the migration with:
```bash
php artisan migrate
```
This will create both the *users* and *password resets* table. 

### User Model
We should add `provider`  and `provider_id` to the list of mass assignable attributes in the `User` model.

*app/User.php*
```php
[...]
protected $fillable = [
        'name', 'email', 'password', 'provider', 'provider_id'
    ];
[...]
```

We can now add authentication to our app with:
```bash 
php artisan make:auth
```

So, if we visit  `http://localhost:3000` you will notice `sign up` and `sign in` links on the nav. Users can now sign up and login to our app via traditional Login. We want them to be able to sign up using Facebook, Twitter and Github.  With [Socialite ](https://github.com/laravel/socialite), adding this functionality is made a breeze since Socialite handles almost all of the boilerplate social authentication code.


### Adding Socialite
To get started with Socialite we'll first require the package 
```bash
composer require laravel/socialite
```

After installing the Socialite library, register the `Laravel\Socialite\SocialiteServiceProvider` in your *config/app.php* configuration file:

```php
'providers' => [
    // Other service providers...

    Laravel\Socialite\SocialiteServiceProvider::class,
],
```

Then add the `Socialite` facade to the `aliases` array in your `app` configuration file:
*config/app*
```php
'Socialite' => Laravel\Socialite\Facades\Socialite::class,
```


### App credentials
The next step is to add the credentials for the OAuth services the application utilizes. These credentials should be placed in your *config/services.php* configuration file. In this app we are going to use Twitter, Github and Facebook.  Let's start by registering our app on twitter so we can get the `client id` and `secret`. We should also set the callbck url for our app.  Visit this [link](https://apps.twitter.com/app/new)  to register your new app. Note, you have to be logged in in order to register an app on Twitter. Set the callback url to `http://localhost:3000/auth/twitter/callback`. Once the app has been created click on the `keys and access tokens` tab to get your *Consumer API Key* and *Consumer API secret*. 

*config/services.php*
```php
[...]
'twitter' => [
        'client_id'     => env('TWITTER_ID'),
        'client_secret' => env('TWITTER_SECRET'),
        'redirect'      => env('TWITTER_URL'),
    ],
[...]
```

Set these variables in your `.env` file
```
TWITTER_ID=Consumer API Key
TWITTER_SECRET=Consumer API secret
TWITTER_URL=callbackurl
```

We will have to repeat this for both Facebook and Github. For facebook, register your app [here](https://developers.facebook.com/apps) then click on `settings` on the sidenav. ![](http://imgur.com/xGnHyHu.png) From the basic settings you will be able to access your `API Key` and `API Secret`.  Also, click the Add Platform button below the settings configuration. Select Website in the platform dialog then enter the website url, in our case it is `http://localhost:3000`. In the App Domains text input, add your domain that matches the one in the URL i.e. `localhost:3000` then save the settings.

Update your *config/services.php* file to take note of Facebook. Note, the values for the `key`,  `secret` and `callback url` should be set in your `.env` file. The `FACEBOOK_URL` in this case will be `http://localhost:3000/auth/facebook/callback` 

*config/services.php*
```php
[...]
'facebook' => [
        'client_id'     => env('FACEBOOK_ID'),
        'client_secret' => env('FACEBOOK_SECRET'),
        'redirect'      => env('FACEBOOK_URL'),
    ],
[...]
```

Visit this [link](https://github.com/settings/applications/new) to create a new app on Github. Enter the app details and you will recieve the API key and secret. Update your *config/services.php* file to take note of Github, similar to how you you did it for Twitter and Facebook.


### Routing
Next, you are ready to authenticate users! You will need two routes: one for redirecting the user to the OAuth provider, and another for receiving the callback from the provider after authentication. 
*app/Http/routes.php*
```php
// Other routes
[...]
// OAuth Routes
Route::get('auth/{provider}', 'Auth\AuthController@redirectToProvider');
Route::get('auth/{provider}/callback', 'Auth\AuthController@handleProviderCallback');
```

### Auth Controller
We will access Socialite using the `Socialite` facade. We will also have to require the `Auth` facade so we can access various `Auth` methods.

*app/Http/Controllers/Auth/AuthController.php*
```php
[...]
use Auth;
use Socialite;
[...]


class AuthController extends Controller
{
    // Some methods which were generated with the app
    [...]
    /**
     * Redirect the user to the provider authentication page.
     *
     * @return Response
     */
    public function redirectToProvider($provider)
    {
        return Socialite::driver($provider)->redirect();
    }

    /**
     * Obtain the user information from provider.  Then login the user.
     *
     * @return Response
     */
    public function handleProviderCallback($provider)
    {
        try {
            $user = Socialite::driver($provider)->user();
        } catch (Exception $e) {
            return redirect('auth/'.$provider);
        }
        
        $authUser = $this->findOrCreateUser($user, $provider);
        Auth::login($authUser, true);
        return redirect($this->redirectTo);
    }
    
    /**
     * Find user based on Oauth details, or create one if none is found.
     */
    public function findOrCreateUser($user, $provider)
    {
        $authUser = User::where('provider_id', $user->id)->first();
        if ($authUser) {
            return $authUser;
        }
        return User::create([
            'name'     => $user->name,
            'email'    => $user->email,
            'provider' => $provider,
            'provider_id' => $user->id
        ]);
    }
    [...]
```

The `redirectToProvider` method takes care of sending the user to the OAuth provider, while the `handleProviderCallback` method will read the incoming request and retrieve the user's information from the provider. Notice we also have a `findOrCreateUser` method which looks up the user object in the database to see if it exists. If the user exists the existing `user object` will be returned. Otherwise, a new user will be created. This prevents users from signing up twice.

It's time to add the links to to the *registration* and *login* views so that users can sign up and login with Facebook, Github and Twitter. In my case I did this right after the *Register* button.

*Resources/Views/Auth/register.blade.php*
```php
[...]
<hr>
<div class="form-group">
    <div class="col-md-6 col-md-offset-4">
        <a href="{{ url('/auth/github') }}" class="btn btn-github"><i class="fa fa-github"></i> Github</a>
        <a href="{{ url('/auth/twitter') }}" class="btn btn-twitter" class="btn btn-twitter"><i class="fa fa-twitter"></i> Twitter</a>
        <a href="{{ url('/auth/facebook') }}" class="btn btn-facebook" class="btn btn-facebook"><i class="fa fa-facebook"></i> Facebook</a>
      </div>
 </div>
```

And this after the *Login* button
*Resources/Views/Auth/login.blade.php*
```php
[...]
<hr>
<div class="form-group">
    <div class="col-md-6 col-md-offset-4">
        <a href="{{ url('/auth/github') }}" class="btn btn-github"><i class="fa fa-github"></i> Github</a>
        <a href="{{ url('/auth/twitter') }}" class="btn btn-twitter" class="btn btn-twitter"><i class="fa fa-twitter"></i> Twitter</a>
        <a href="{{ url('/auth/facebook') }}" class="btn btn-facebook" class="btn btn-facebook"><i class="fa fa-facebook"></i> Facebook</a>
      </div>
 </div>
```

And there you have it, users can now sign up with various social providers. Socialite is not limited to Twitter, Facebook and Github as it also supports LinkedIn, Google and Bitbucket. I hope the tutorial was helpful.





