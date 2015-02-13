# Laravel Facebook SDK

[![Build Status](https://img.shields.io/travis/SammyK/LaravelFacebookSdk.svg)](https://travis-ci.org/SammyK/LaravelFacebookSdk)
[![Latest Stable Version](https://img.shields.io/badge/Development%20Version-2.0.0-orange.svg)](https://packagist.org/packages/sammyk/laravel-facebook-sdk)
[![Total Downloads](https://img.shields.io/packagist/dt/sammyk/laravel-facebook-sdk.svg)](https://packagist.org/packages/sammyk/laravel-facebook-sdk)
[![License](https://img.shields.io/badge/license-MIT-lightgrey.svg)](https://github.com/SammyK/LaravelFacebookSdk/blob/master/LICENSE)


A fully unit-tested package for easily integrating the [Facebook SDK v4.1](https://github.com/facebook/facebook-php-sdk-v4/tree/master) into Laravel 5.

> **This is package for Laravel 5!** For Laravel 4.2, [see the 1.2 branch](https://github.com/SammyK/LaravelFacebookSdk/tree/1.2). :+1:

----

- [Installation](#installation)
- [Facebook Login](#facebook-login)
- [Saving Data From Facebook In The Database](#saving-data-from-facebook-in-the-database)
- [Logging The User Into Laravel](#logging-the-user-into-laravel)
- [Error Handling](#error-handling)
- [Testing](#testing)
- [Contributing](#contributing)
- [Credits](#credits)
- [License](#license)


## Shouldn't I just use Laravel Socialite?

Laravel 5 comes with support for [Socialite](http://laravel.com/docs/5.0/authentication#social-authentication) which allows you to authenticate with OAuth 2.0 providers. Facebook Login uses OAuth 2.0 and therefore Socialite supports Facebook Login.

If all you need is to authenticate an app and grab a user access token to pull basic data on a user, then Socialite or The PHP League's [Facebook OAuth Client](https://github.com/thephpleague/oauth2-facebook) should suffice for your needs.

But if you need any of the following features, you'll want to tie in the Facebook PHP SDK with this package:

- Obtaining an access token from the signed request in:
    - The cookie set by the Facebook JavaScript SDK
    - The `signed_request` param `POST`'ed to an app canvas
    - The `signed_request` param `POST`'ed to a Facebook page tab
- Photo or video uploads
- Batch requests
- Easy pagination
- Getting Graph data returned as collections


## Installation

Add the Laravel Facebook SDK package to your `composer.json` file.

```json
{
    "require": {
        "sammyk/laravel-facebook-sdk": "~2.0@dev",
        "facebook/php-sdk-v4": "~4.1.0@dev"
    }
}
```

> **Note:** The Facebook PHP SDK v4.1 is still in dev mode but has reached a feature-freeze until it is tagged as stable so there shouldn't be any breaking changes. :) But because it's in dev mode you'll need to require it explicitly in your require using the `@dev` minimum stability flag since [composer won't pull in a dev mode dependency of a dependency](https://getcomposer.org/doc/faqs/why-can%27t-composer-load-repositories-recursively.md).


### Service Provider

In your app config, add the `LaravelFacebookSdkServiceProvider` to the providers array.

```php
'providers' => [
    'SammyK\LaravelFacebookSdk\LaravelFacebookSdkServiceProvider',
    ];
```


### Facade (optional)

If you want to make use of the facade, add it to the aliases array in your app config.

```php
'aliases' => [
    'Facebook' => 'SammyK\LaravelFacebookSdk\FacebookFacade',
    ];
```

But there are [much better ways](#ioc-container) to use this package that [don't use facades](http://programmingarehard.com/2014/01/11/stop-using-facades.html).


### IoC container

The IoC container will automatically resolve the `LaravelFacebookSdk` dependencies for you. You can grab an instance of `LaravelFacebookSdk` from the IoC container in a number of ways.

```php
// Directly from App::make();
$fb = App::make('SammyK\LaravelFacebookSdk\LaravelFacebookSdk');

// From a constructor
class FooClass {
    public function __construct(SammyK\LaravelFacebookSdk\LaravelFacebookSdk $fb) {
       // . . .
    }
}

// From a method
class BarClass {
    public function barMethod(SammyK\LaravelFacebookSdk\LaravelFacebookSdk $fb) {
       // . . .
    }
}

// Or even a closure
Route::get('/facebook/login', function(SammyK\LaravelFacebookSdk\LaravelFacebookSdk $fb) {
    // . . .
});
```


### Configuration File

After [creating an app in Facebook](https://developers.facebook.com/apps), you'll need to provide the app ID and secret. First publish the configuration file.

```bash
$ php artisan vendor:publish --provider="sammyk/laravel-facebook-sdk" --tag="config"
```

> **Where's the file?** Laravel 5 will publish the config file to `config/laravel-facebook-sdk.php`.


#### Required config values

You'll need to update the `app_id` and `app_secret` values in the config file with [your app ID and secret](https://developers.facebook.com/apps).


### Syncing Graph nodes with Laravel models

If you have a `facebook_user_id` column in your user's table, you can add the `SyncableGraphNodeTrait` to your `User` model to have the user node from the Graph API automatically sync with your model.

```php
class User extends Eloquent implements UserInterface {
    use SammyK\LaravelFacebookSdk\SyncableGraphNodeTrait;
    
    protected static $graph_node_field_aliases = [
        'id' => 'facebook_user_id',
    ];
}
```

More info on [saving data from Facebook in the database](#saving-data-from-facebook-in-the-database).


## User Login From Redirect Example

Here's a full example of how you might log a user into your app using the [redirect method](#login-from-redirect).

This example also demonstrates how to [exchange a short-lived access token with a long-lived access token](https://www.sammyk.me/access-token-handling-best-practices-in-facebook-php-sdk-v4) and save the user to your `users` table if the entry doesn't exist.

Finally it will log the user in using Laravel's built-in user authentication.

``` php
// Generate a login URL
Route::get('/facebook/login', function(SammyK\LaravelFacebookSdk\LaravelFacebookSdk $fb)
{
    // Send an array of permissions to request
    $login_url = $fb->getLoginUrl(['email']);

    // Obviously you'd do this in blade :)
    echo '<a href="' . $login_url . '">Login with Facebook</a>';
});

// Endpoint that is redirected to after an authentication attempt
Route::get('/facebook/callback', function(SammyK\LaravelFacebookSdk\LaravelFacebookSdk $fb)
{
    // Obtain an access token.
    try {
        $token = $fb->getAccessTokenFromRedirect();
    } catch (Facebook\Exceptions\FacebookSDKException $e) {
        dd($e->getMessage());
    }

    // Access token will be null if the user denied the request
    // or if someone just hit this URL outside of the OAuth flow.
    if (! $token) {
        // Get the redirect helper
        $helper = $fb->getRedirectLoginHelper();

        if (! $helper->getError()) {
            abort(403, 'Unauthorized action.');
        }

        // User denied the request
        dd(
            $helper->getError(),
            $helper->getErrorCode(),
            $helper->getErrorReason(),
            $helper->getErrorDescription()
        );
    }

    if (! $token->isLongLived()) {
        // OAuth 2.0 client handler
        $oauth_client = $fb->getOAuth2Client();

        // Extend the access token.
        try {
            $token = $oauth_client->getLongLivedAccessToken($token);
        } catch (Facebook\Exceptions\FacebookSDKException $e) {
            dd($e->getMessage());
        }
    }

    $fb->setDefaultAccessToken($token);

    // Save for later
    Session::put('fb_user_access_token', (string) $token);

    // Get basic info on the user from Facebook.
    try {
        $response = $fb->get('/me?fields=id,name,email');
    } catch (Facebook\Exceptions\FacebookSDKException $e) {
        dd($e->getMessage());
    }

    // Convert the response to a `Facebook/GraphNodes/GraphUser` collection
    $facebook_user = $response->getGraphUser();

    // Create the user if it does not exist or update the existing entry.
    // This will only work if you've added the SyncableGraphNodeTrait to your User model.
    $user = App\User::createOrUpdateGraphNode($facebook_user);

    // Log the user into Laravel
    Auth::login($user);

    return redirect('/')->with('message', 'Successfully logged in with Facebook');
});
```


## Facebook Login

When we say "log in with Facebook", we really mean "obtain a user access token to make calls to the Graph API on behalf of the user." This is done through Facebook via [OAuth 2.0](http://oauth.net/2/). There are a number of ways to log a user in with Facebook using what the Facbeook PHP SDK calls "[helpers](https://developers.facebook.com/docs/php/reference#helpers)".

The four supported login methods are:

1. [Login From Redirect](#login-from-redirect) (OAuth 2.0)
2. [Login From JavaScript](#login-from-javascript) (with JS SDK cookie)
3. [Login From App Canvas](#login-from-app-canvas) (with signed request)
4. [Login From Page Tab](#login-from-page-tab) (with signed request)


### Login From Redirect

One of the most common ways to log a user into your app is by using a redirect URL.

The idea is that you generate a unique URL that the user clicks on. Once the user clicks the link they will be redirected to Facebook asking them to grant any permissions your app is requesting. Once the user responds, Facebook will redirect the user back to a callback URL that you specify with either a successful response or an error response.

The redirect helper can be obtained using the SDK's `getRedirectLoginHelper()` method.


#### Generating a login URL

You can get a login URL just like you you do with the Facebook PHP SDK v4.1.

```php
Route::get('/facebook/login', function(SammyK\LaravelFacebookSdk\LaravelFacebookSdk $fb) {
    $login_link = $fb
            ->getRedirectLoginHelper()
            ->getLoginUrl('https://exmaple.com/facebook/callback', ['email', 'user_events']);
    
    echo '<a href="' . $login_link . '">Log in with Facebook</a>';
});

```

But if you set the `default_redirect_uri` callback URL in the config file, you can use the `getLoginUrl()` wrapper method which will default the callback URL (`default_redirect_uri`) and permission scope (`default_scope`) to whatever you set in the config file.

```php
$login_link = $fb->getLoginUrl();
```

Alternatively you can pass the permissions and a custom callback URL to the wrapper to overwrite the default config.

> **Note:** Since the list of permissions sometimes changes but the callback URL usually stays the same, the permissions array is the first argument in the `getLoginUrl()` wrapper method which is the reverse of the SDK's method `getRedirectLoginHelper()->getLoginUrl($url, $permissions)`.

```php
$login_link = $fb->getLoginUrl(['email', 'user_status'], 'https://exmaple.com/facebook/callback');
// Or, if you want to default to the callback URL set in the config
$login_link = $fb->getLoginUrl(['email', 'user_status']);
```


#### Obtaining an access token from a callback URL

After the user has clicked on the login link from above and confirmed or denied the app permission requests, they will be redirected to the specified callback URL.

The standard "SDK" way to obtain an access token on the callback URL is as follows:

```php
Route::get('/facebook/callback', function(SammyK\LaravelFacebookSdk\LaravelFacebookSdk $fb) {
    try {
        $token = $fb
            ->getRedirectLoginHelper()
            ->getAccessToken();
    } catch (Facebook\Exceptions\FacebookSDKException $e) {
        // Failed to obtain access token
        dd($e->getMessage());
    }
});
```

There is a wrapper method for `getRedirectLoginHelper()->getAccessToken()` in LaravelFacebookSdk called `getAccessTokenFromRedirect()` that defaults the callback URL to whatever the current URL is.

```php
Route::get('/facebook/callback', function(SammyK\LaravelFacebookSdk\LaravelFacebookSdk $fb) {
    try {
        $token = $fb->getAccessTokenFromRedirect();
    } catch (Facebook\Exceptions\FacebookSDKException $e) {
        // Failed to obtain access token
        dd($e->getMessage());
    }
    
    // $token will be null if the user denied the request
    if (! $token) {
        // User denied the request
    }
});
```


### Login From JavaScript

If you're using the [JavaScript SDK](https://developers.facebook.com/docs/javascript), you can obtain an access token from the cookie set by the JavaScript SDK.

By default the JavaScript SDK will not set a cookie, so you have to explicitly enable it with `cookie: true` when you `init()` the SDK.

```javascript
FB.init({
  appId      : 'your-app-id',
  cookie     : true,
  version    : 'v2.2'
});
```

After you have logged a user in with the JavaScript SDK using [`FB.login()`](https://developers.facebook.com/docs/reference/javascript/FB.login), you can obtain a user access token from the signed request that is stored in the cookie that was set by the JavaScript SDK.

```php
Route::get('/facebook/javascript', function(SammyK\LaravelFacebookSdk\LaravelFacebookSdk $fb) {
    try {
        $token = $fb->getJavaScriptHelper()->getAccessToken();
    } catch (Facebook\Exceptions\FacebookSDKException $e) {
        // Failed to obtain access token
        dd($e->getMessage());
    }

    // $token will be null if no cookie was set or no OAuth data
    // was found in the cookie's signed request data
    if (! $token) {
        // User hasn't logged in using the JS SDK yet
    }
});
```


### Login From App Canvas

If your app lives within the context of a Facebook app canvas, you can obtain an access token from the signed request that is `POST`'ed to your app on the first page load.

> **Note:** The canvas helper only obtains an existing access token from the signed request data received from Facebook. If the user visiting your app has not authorized your app yet or their access token has expired, the `getAccessToken()` method will return `null`. In that case you'll need to log the user in with either [a redirect](#login-from-redirect) or [JavaScript](#login-from-javascript).

Use the SDK's canvas helper to obtain the access token from the signed request data.

```php
Route::get('/facebook/canvas', function(SammyK\LaravelFacebookSdk\LaravelFacebookSdk $fb) {
    try {
        $token = $fb->getCanvasHelper()->getAccessToken();
    } catch (Facebook\Exceptions\FacebookSDKException $e) {
        // Failed to obtain access token
        dd($e->getMessage());
    }

    // $token will be null if the user hasn't authenticated your app yet
    if (! $token) {
        // . . .
    }
});
```


### Login From Page Tab

If your app lives within the context of a Facebook Page tab, that is the same as an app canvas and the "Login From App Canvas" method will also work to obtain an access token. But a Page tab also has additional data in the signed request.

The SDK provides a Page tab helper to obtain an access token from the signed request data within the context of a Page tab.

```php
Route::get('/facebook/page-tab', function(SammyK\LaravelFacebookSdk\LaravelFacebookSdk $fb) {
    try {
        $token = $fb->getPageTabHelper()->getAccessToken();
    } catch (Facebook\Exceptions\FacebookSDKException $e) {
        // Failed to obtain access token
        dd($e->getMessage());
    }

    // $token will be null if the user hasn't authenticated your app yet
    if (! $token) {
        // . . .
    }
});
```

## Other authorization requests

Facebook supports two other types of authorization URL's - rerequests and re-authentications.

### Rerequests

[Rerequests](https://developers.facebook.com/docs/facebook-login/manually-build-a-login-flow#reaskperms) (or re-requests?) ask the user again for permissions they have previously declined. It's important to use a rerequest URL for this instead of just redirecting them with the normal log in link because:

> Once someone has declined a permission, the Login Dialog will not re-ask them for it unless you explicitly tell the dialog you're re-asking for a declined permission. - [Facebook Documentation](https://developers.facebook.com/docs/facebook-login/manually-build-a-login-flow#reaskperms)

You can generate a rerequest URL using the `getReRequestUrl()` method.

```php
$rerequest_link = $fb->getReRequestUrl(['email'], 'https://exmaple.com/facebook/login');
// Or, if you want to default to the callback URL set in the config
$rerequest_link = $fb->getReRequestUrl(['email']);
```


### Re-authentications

[Re-authentications](https://developers.facebook.com/docs/facebook-login/reauthentication) force a user to confirm their identity by asking them to enter their Facebook account password again. This is useful for adding another layer of security before changing or view sensitive data on your web app.

You can generate a re-authentication URL using the `getReAuthenticationUrl()` method.

```php
$re_authentication_link = $fb->getReAuthenticationUrl(['email'], 'https://exmaple.com/facebook/login');
// Or, if you want to default to the callback URL set in the config
$re_authentication_link = $fb->getReAuthenticationUrl(['email']);
// Or without permissions
$re_authentication_link = $fb->getReAuthenticationUrl();
```


## Saving the Access Token

In most cases you won't need to save the access token to a database unless you plan on making requests to the Graph API on behalf of the user when they are not browsing your app (like a 3AM CRON job for example).

After you obtain an access token, you can store it in a session to be used for subsequent requests.

```php
Session::put('facebook_access_token', (string) $token);
```

Then in each script that makes calls to the Graph API you can pull the token out of the session and set it as the default.

```php
$token = Session::get('facebook_access_token');
$fb->setDefaultAccessToken($token);
```


## Saving data from Facebook in the database

Saving data received from the Graph API to a database can sometimes be a tedious endeavor. Since the Graph API returns data in a predictable format, the `SyncableGraphNodeTrait` can make saving the data to a database a painless process.

Any Eloquent model that implements the `SyncableGraphNodeTrait` will have the `createOrUpdateGraphNode()` method applied to it. This method really makes it easy to take data that was returned directly from Facebook and create or update it in the local database.

```php
use SammyK\LaravelFacebookSdk\SyncableGraphNodeTrait;

class Event extends Eloquent {
    use SyncableGraphNodeTrait;
}
```

For example if you have an Eloquent model named `Event`, here's how you might grab a specific event from the Graph API and insert it into the database as a new entry or update an existing entry with the new info.

```php
$response = $fb->get('/some-event-id?fields=id,name');
$facebook_event = $response->getGraphObject();

// A method called createOrUpdateGraphNode() on the `Event` eloquent model
// will create the event if it does not exist or it will update the existing
// record based on the ID from Facebook.
$event = Event::createOrUpdateGraphNode($facebook_event);
```

> **Heads up!** The Facebook PHP SDK calls nodes from the Graph API `GraphObject`'s. This is not a good name for them since the Graph API calls them nodes. This will [most likely be renamed](https://github.com/facebook/facebook-php-sdk-v4/issues/320) to `GraphNode` in a breaking change pretty soon.

The `createOrUpdateGraphNode()` will automatically map the returned field names to the column names in your database. If, for example, your column names on the `events` table don't match the field names for an [Event](https://developers.facebook.com/docs/graph-api/reference/event) node, you can [map the fields](#field-mapping).


### Field Mapping

Since the names of the columns in your database might not match the names of the fields of the Graph nodes, you can map the field names in your `User` model using the `$graph_node_field_aliases` static variable.

The *keys* of the array are the names of the fields on the Graph node. The *values* of the array are the names of the columns in the local database.

```php
use SammyK\LaravelFacebookSdk\SyncableGraphNodeTrait;

class User extends Eloquent implements UserInterface
{
    use SyncableGraphNodeTrait;
    
    protected static $graph_node_field_aliases = [
        'id' => 'facebook_user_id',
        'name' => 'full_name',
        'graph_node_field_name' => 'database_column_name',
    ];
}
```


## Logging The User Into Laravel

The Laravel Facebook SDK makes it easy to log a user in with Laravel's built-in authentication driver.


### Updating The Users Table

In order to get Facebook authentication working with Laravel's built-in authentication, you'll need to store the Facebook user's ID in your user's table.

Naturally you'll need to create a column for every other piece of information you want to keep about the user.

You can store the access token in the database if you need to make requests on behalf of the user when they are not browsing your app (like a 3AM cron job). But in general you won't need to store the access token in the database.

You'll need to generate a migration to modify your `users` table and add any new columns.

> **Note:** Make sure to change `<name-of-users-table>` to the name of your user table.

```bash
$ php artisan make:migration add_facebook_columns_to_users_table --table="<name-of-users-table>"
```

Now update the migration file to include the new fields you want to save on the user. At minimum you'll need to save the Facebook user ID.

```php
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Database\Migrations\Migration;

class AddFacebookColumnsToUsersTable extends Migration
{
    public function up()
    {
        Schema::table('users', function(Blueprint $table)
        {
            // If the primary id in your you user table is different than the Facebook id
            // Make sure it's an unsigned() bigInteger()
            $table->bigInteger('facebook_user_id')->unsigned()->index();
            // Normally you won't need to store the access token in the database
            $table->string('access_token')->nullable();
        });
    }

    public function down()
    {
        Schema::table('users', function(Blueprint $table)
        {
            $table->dropColumn(
                'facebook_user_id',
                'access_token'
            );
        });
    }
}
```

Don't forget to run the migration.

```bash
$ php artisan migrate
```

If you plan on using the Facebook user ID as the primary key, make sure you have a column called `id` that is an unsigned big integer and indexed. If you are storing the Facebook ID in a different field, make sure that field exists in the database and make sure to [map to it in your model](#field-mapping) to your custom id name.

If you're using the Eloquent ORM and storing the access token in the database, make sure to hide the `access_token` field from possible exposure in your `User` model.

Don't forget to add the [`SyncableGraphNodeTrait`](#saving-data-from-facebook-in-the-database) to your user model so you can sync your model with data returned from the Graph API.

```php
# User.php
use SammyK\LaravelFacebookSdk\SyncableGraphNodeTrait;

class User extends Eloquent implements UserInterface {
    use SyncableGraphNodeTrait;

    protected $hidden = ['access_token'];
}
```


### Logging the user into Laravel

After the user has logged in with Facebook and you've obtained the user ID from the Graph API, you can log the user into Laravel by passing the logged in user's `User` model to the `Auth::login()` method.

```php
class FacebookController {
    public function getUserInfo(SammyK\LaravelFacebookSdk\LaravelFacebookSdk $fb) {
       try {
           $response = $fb->get('/me?fields=id,name,email');
       } catch (Facebook\Exceptions\FacebookSDKException $e) {
           dd($e->getMessage());
       }
       
       // Convert the response to a `Facebook/GraphNodes/GraphUser` collection
       $facebook_user = $response->getGraphUser();
       
       // Create the user if it does not exist or update the existing entry.
       // This will only work if you've added the SyncableGraphNodeTrait to your User model.
       $user = App\User::createOrUpdateGraphNode($facebook_user);
       
       // Log the user into Laravel
       Auth::login($user);
    }
}
```


## Error Handling

The Facebook PHP SDK throws `Facebook\Exceptions\FacebookSDKException` exceptions. Whenever there is an error response from Graph, the SDK will throw a `Facebook\Exceptions\FacebookResponseException` which extends from `Facebook\Exceptions\FacebookSDKException`. If a `Facebook\Exceptions\FacebookResponseException` is thrown you can grab a specific exception related to the error from the `getPrevious()` method.

```php
try {
    // Stuffs here
} catch (Facebook\Exceptions\FacebookResponseException $e) {
    $graphError = $e->getPrevious();
    echo 'Graph API Error: ' . $e->getMessage();
    echo ', Graph error code: ' . $graphError->getCode();
    exit;
} catch (Facebook\Exceptions\FacebookSDKException $e) {
    echo 'SDK Error: ' . $e->getMessage();
    exit;
}
```

The LaravelFacebookSdk does not throw any custom exceptions.


## Testing

The tests are written with `phpunit`. You can run the tests from the root of the project directory with the following command.

``` bash
$ ./vendor/bin/phpunit
```


## Contributing

Please see [CONTRIBUTING](https://github.com/SammyK/LaravelFacebookSdk/blob/master/CONTRIBUTING.md) for details.


## Credits

This package is maintained by [Sammy Kaye Powers](https://github.com/SammyK). See a [full list of contributors](https://github.com/SammyK/LaravelFacebookSdk/contributors).


## License

The MIT License (MIT). Please see [License File](https://github.com/SammyK/LaravelFacebookSdk/blob/master/LICENSE) for more information.
