# laravel-jwtredis

[![GitHub license](https://img.shields.io/badge/license-MIT-brightgreen.svg?style=flat-square)](https://raw.githubusercontent.com/sametsahindogan/laravel-jwtredis/master/LICENSE)

> This package allows JWT-authenticated users to be stored and management in Redis with
their roles, permissions, statuses and anything you want.

<a><img src="https://cdn.auth0.com/blog/jwtalgos/logo.png" width="80"></a>
<a><img src="https://chris.lu/upload/images/redis.png" width="80"></a>

>Also this package have an observer for listening and updating  to your user model 
on Redis.This observer is triggered `when you assign roles & permissions to user, or update
and delete to your user` model.

## Requirements

This package work with together [tymondesigns/jwt-auth](https://github.com/tymondesigns/jwt-auth) and [spatie/laravel-permission](https://github.com/spatie/laravel-permission) package under the hood.

![#](https://placehold.it/15/f03c15/000000?text=+) `Make sure to install and configure these dependencies. You must publish, migrate etc. all packages.` ![#](https://placehold.it/15/f03c15/000000?text=+)
 
- [nrk/predis](https://github.com/nrk/predis) **>= 1.1** (**Recommended 1.1**)
- [tymondesigns/jwt-auth](https://github.com/tymondesigns/jwt-auth) **>= 1.0** (**Recommended 1.0.x**)
- [spatie/laravel-permission](https://github.com/spatie/laravel-permission) **>= 3.3** (**Recommended 3.3**)
- [sametsahindogan/response-object-creator](https://github.com/sametsahindogan/response-object-creator) **>= 1.0.x** (**Recommended 1.0.2**)

## Installation

```bash
composer require sametsahindogan/laravel-jwtredis
```
Once this has finished, you will need to add&change these values in `.env` file:
```dotenv
CACHE_DRIVER=redis
REDIS_CLIENT=predis
```
Next, you will need to change the `guards` and `providers` arrays in your `config/auth.php` config as follows:
```php
'guards' => [
        'api' => [
            'driver' => 'jwt_redis_guard',
            'provider' => 'users'
        ],
    ],

'providers' => [
        'users' => [
            'driver' => 'jwt_redis_user_provider',
            'model' =>  App\User::class, /* Your User Model */
        ],
    ],
```
This package uses auto-discovery to register the service provider but if you'd rather do it manually, the service provider is: add to `providers` array in your `config/app.php` config as follows:
```php
Sametsahindogan\JWTRedis\JWTRedisServiceProvider::class,
```
Next, also in the `config/app.php` config file, under the `aliases` array, you may want to add the `RedisCache` facade.
```php
'RedisCache' => \Sametsahindogan\JWTRedis\Facades\RedisCache::class,
```
Finally, you will want to publish the config using the following command:
```bash
php artisan vendor:publish --provider='Sametsahindogan\JWTRedis\JWTRedisServiceProvider'
```

## Configurations

When everything is complete, don't forget to add this Trait to your user model.
```php
use JWTRedisHasRoles;
```
You need to add `$routeMiddleware` array in `app/Http/Kernel.php`
```php
'auth'               => \Sametsahindogan\JWTRedis\Http\Middleware\Authenticate::class,
'refreshable'        => \Sametsahindogan\JWTRedis\Http\Middleware\Refreshable::class,
'role'               => \Sametsahindogan\JWTRedis\Http\Middleware\RoleMiddleware::class,
'permission'         => \Sametsahindogan\JWTRedis\Http\Middleware\PermissionMiddleware::class,
'role_or_permission' => \Sametsahindogan\JWTRedis\Http\Middleware\RoleOrPermissionMiddleware::class,
```

## Usage

 You do not have any instructions for use. This package only affects the background, functions in an almost identical way to Laravel session authentication, with a few exceptions. `All you have to do is change your middleware.(I mention this below)` You can use Laravel's Auth facade,
Tymon's JWTAuth facade and all [spatie/laravel-permission](https://github.com/spatie/laravel-permission) package methods as usual.<br>

* For user authentication by token; <br>
( Use this middleware if the user's identity is not important. This middleware only checks if Token is valid. Doesn't send to any query to any database.)
```php
Route::get("/example", "ExampleController@example")->middleware('auth');
```
* To check user authorization, you need to this one of these middlewares;<br>
( Use this middleware if the user's identity is important. This middlewares fetch user from Redis and mark as authorized 
to Laravel's Request object. And you will reach all default Auth facade's methods you want. Just call Laravel's Auth facades.)
```php
Route::get("/example", "ExampleController@example")->middleware('role:admin|user');
Route::get("/example", "ExampleController@example")->middleware('permissions:get-user|set-user');
Route::get("/example", "ExampleController@example")->middleware('role_or_permission:admin|get-user');
```
* To refresh the token, you can add the `refreshable` middleware to the required route. You don't need to take any action on the controller of this route;<br>
( Also this middleware can refreshes user from Redis if necessary. )
```php
Route::get("/example", "ExampleController@example")->middleware('refreshable');
```
<br>

`If you want to do different things, you can override those mentioned middlewares.`

**After using it as above, every authorization you made in your 
application, such as `Auth::user()` or `$user->can('permission')`, is always checked from Redis, not from the database.**

## Options

You can customize some options in that package `config/jwtredis.php` file.

You can set your user model.
```php
'user_model' =>  \App\User::class,
```
If you want to use your custom observer class, you can also change this or inherit this class.
```php
'observer' => \Sametsahindogan\JWTRedis\Observers\UserRedisObserver::class,
```
If observer events queue option is true, model's events are dispatched in the queue.
*  Don't forget to set Queue Driver for Redis and run Laravel Queue Worker.
```php
'observer_events_queue' => true,
```

If you want to store user in Redis until JWT expire time, this option must be true.
```php
'redis_ttl_jwt' => true,
```
- If you don't want to store user in Redis until JWT expire time, you can set this value as minute.
```php
    'redis_ttl' => 60
```
If check banned user option is true, user status checked by necessary middlewares.
```php
'check_banned_user' => false,
```
- If you want to check banned user; you can set your own table column title and status values.
```php
    'status_column_title' => 'status',
    'banned_statuses' => ['banned', 'deactivate']
```
You can add this array to your own relations and anything you want to store in Redis.
```php
'cache_relations' => [
        'roles.permissions',
        'permissions'
    ],
```
You can customize error code,message,title for your application.
```php
 'errors' => [
        'TokenNotProvidedException' => [
            'title' => 'Your custom title',
            'message' => 'Your custom error message.',
            'code' => 99999
        ]
]
```

## Example Project

Here is an [example usage](https://github.com/sametsahindogan/laravel-jwtredis-example) using laravel-jwtredis. You can examine in detail.

## License
MIT © [Samet Sahindogan](https://github.com/sametsahindogan/laravel-jwtredis/blob/master/LICENSE)
