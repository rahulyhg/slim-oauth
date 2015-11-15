# Slim Framework OAuth Middleware

[![Code Climate](https://codeclimate.com/github/slimphp-api/slim-oauth/badges/gpa.svg)](https://codeclimate.com/github/slimphp-api/slim-oauth)


This repository contains a Slim Framework OAuth middleware.

Enables you to authenticate using various OAuth providers.

The middleware allows registration with various oauth services and uses a user service to register/retrieve the user details.
After registration/authentication it responds with a Authorization header which it expects to be returned as is to authorise further requests.
It's up to the supplied user service how this is accomplished.

## Installation

Via Composer

```bash
$ composer require slimphp-api/slim-oauth
```

Requires Slim 3.0.0 or newer.

## Usage

```php
<?php
use Slim\App;
use SlimApi\OAuth\OAuthFactory;
use SlimApi\OAuth\OAuthMiddleware;

$app = new App();

$container = $app->getContainer();

// these should all probably be in some configuration class
$container['oAuthCreds'] = [
    'github' => [
        'key'       => 'abc',
        'secret'    => '123',
    ]
];

$container[SlimApi\OAuth\OAuthFactory::class]         = function($container)
{
    return new OAuthFactory($container->get('oAuthCreds'));
};

$container[SlimApi\OAuth\UserServiceInterface::class] = function($container)
{
    //user service should implement SlimApi\OAuth\UserServiceInterface
    //user model should have a token variable to hold the random token sent to the client
    return new Foo\Service\UserService($container->get('Foo\Model\User'));
};

$container[SlimApi\OAuth\OAuthMiddleware::class]      = function($container)
{
    return new OAuthMiddleware($container->get('SlimApi\OAuth\OAuthFactory'), $container->get('SlimApi\OAuth\UserServiceInterface'));
};

$app->add($container->get('SlimApi\OAuth\OAuthMiddleware'));

$app->run();
```

Example user service

```php
<?php
namespace Foo\Service;

use SlimApi\OAuth\UserServiceInterface;
use OAuth\Common\Service\ServiceInterface;

class UserService implements UserServiceInterface {

    public function __construct($userModel)
    {
        $this->userModel = $userModel;
    }

    public function createUser(ServiceInterface $service)
    {
        // request the user information from github
        // could go further with this and check org/team membership
        $user = json_decode($service->request('user'), true);

        // create and save a new user
        $model = new $this->userModel([
            'oauth_token' => $service->getStorage()->retrieveAccessToken('GitHub')->getAccessToken(),
            'token'       => 'randomstringj0' // this isn't really random, but it should be!
        ]);
        $model->save();
        return $model;
    }

    public function findOrNew($authToken)
    {
        // retrieve the user by the authToken provided
        // this could also be from some fast access redis db
        $users = $this->userModel->byToken($authToken)->get();
        $user = $users->first();
        // or return a blank entry if it doesn't exist
        return ($user ?: new $this->userModel);
    }
}
```

Once it's all configured redirecting the user to `https://domain/auth/<oauthtype>?return=<https://post.authentication/frontend>`
where oauthtype is the service to authentication ie github and the return url parameter is where you want the user redirected to AFTER authentication.

## Process cycle

```
Client (https://www.example.com) requires the user to register/authenticate
-> redirects to https://api.example.com/auth/github?return=https://www.example.com/authenticated
-> api redirects to GitHub to authenticate
-> GitHub asks user to verify
-> GitHub redirects back to https://api.example.com/auth/github/callback with a temp code in the url
-> api exchanges temp code for permanent token
-> api asks user service to verify/store user and details and return user object (must have token param)
-> api redirects back to client https://www.example.com/authenticated with an Authorization header `'token '.$user->token`
-> client adds Authorization header to all subsequent requests
-> api retrieves user object by Authorization token to check existence
```

## Credits

- [Gabriel Baker](https://github.com/gabriel403)

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.
