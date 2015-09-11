# psr7-middlewares

[![Build Status](https://travis-ci.org/oscarotero/psr7-middlewares.svg)](https://travis-ci.org/oscarotero/psr7-middlewares)
[![Scrutinizer Code Quality](https://scrutinizer-ci.com/g/oscarotero/psr7-middlewares/badges/quality-score.png?b=master)](https://scrutinizer-ci.com/g/oscarotero/psr7-middlewares/?branch=master)

Collection of PSR-7 middlewares

## Requirements

* PHP >= 5.5
* A PSR-7 HTTP Message implementation, for example [zend-diactoros](https://github.com/zendframework/zend-diactoros)
* A PSR-7 middleware dispatcher. For example [Relay](https://github.com/relayphp/Relay.Relay) or any other similar.

## Usage example:

```php
use Psr7Middlewares\Middleware;

use Relay\RelayBuilder;
use Zend\Diactoros\Response;
use Zend\Diactoros\ServerRequestFactory;
use Zend\Diactoros\Stream;

//Set a stream factory used by some middlewares
Middleware::setStreamFactory(function ($file, $mode) {
    return new Stream($file, $mode);
});

//Create a relay dispatcher and add some middlewares:
$relay = new RelayBuilder();

$dispatcher = $relay->newInstance([
    
    //Handle errors
    Middleware::errorHandler()
        ->handler('error_handler_function')
        ->catchExceptions(true)

    //Remove the path prefix
    Middleware::BasePath('/my-site/web'),

    //Digest authentication
    Middleware::DigestAuthentication()
        ->users([
            'username' => 'password'
        ]),

    //Get the client ip
    Middleware::ClientIp(),

    //Allow only some ips
    Middleware::Firewall()
        ->trusted(['127.0.0.*']),

    //Detects the user preferred language
    Middleware::LanguageNegotiator()
        ->languages(['gl', 'es', 'en']),

    //Detects the format
    Middleware::FormatNegotiator(),

    //Execute fast route
    Middleware::FastRoute()
        ->router($app->get('dispatcher')),

    //Minify the result
    Middleware::Minify()

    //Saves the response in a file
    Middleware::saveResponse()
        ->storage('app/public')
]);

$response = $dispatcher(ServerRequestFactory::fromGlobals(), new Response());
```

## Available middlewares

* [AuraRouter](#aurarouter)
* [AuraSession](#aurasession)
* [BasePath](#basepath)
* [BasicAuthentication](#basicauthentication)
* [Cache](#cache)
* [ClientIp](#clientip)
* [DigestAuthentication](#digestauthentication)
* [ErrorHandler](#errorhandler)
* [FastRoute](#fastroute)
* [Firewall](#firewall)
* [FormatNegotiation](#formatnegotiation)
* [LanguageNegotiation](#languagenegotiation)
* [Minify](#minify)
* [SaveResponse](#saveresponse)
* [TrailingSlash](#trailingslash)

### AuraRouter

To use [Aura.Router](https://github.com/auraphp/Aura.Router) as a middleware. You need to use the 3.x version, compatible with psr-7:

```php
use Psr7Middlewares\Middleware;
use Psr7Middlewares\Middleware\AuraRouter;
use Aura\Router\RouterContainer;

//Create the router
$routerContainer = new RouterContainer();

$map = $routerContainer->getMap();

$map->get('hello', '/hello/{name}', function ($request, $response, $myApp) {

    //The route values are stored into parameters
    $name = $request->getAttribute('name');

    //You can get also the route instance
    $route = AuraRouter::getRoute($request);

    //Write directly in the body's response
    $response->getBody()->write('Hello '.$name);

    //or echo the output (it will be captured and passed to body stream)
    echo 'Hello world';

    //or return a string
    return 'Hello world';

    //or return a new response
    return $response->withStatus(200);
});

//Add to the dispatcher
$dispatcher = $relay->getInstance([

    Middleware::AuraRouter()
        ->router($routerContainer) //Instance of Aura\Router\RouterContainer
        ->arguments($myApp) //(optional) append more arguments to the controller
]);
```

### AuraSession

Creates a new [Aura.Session](https://github.com/auraphp/Aura.Session) instance with the request.

```php
use Psr7Middlewares\Middleware;
use Psr7Middlewares\Middleware\AuraSession;

$dispatcher = $relay->getInstance([

    Middleware::AuraSession(),
        ->factory($sessionFactory) //(optional) Intance of Aura\Session\SessionFactory
        ->name('my-session-name'), //(optional) custom session name

    function ($request, $reponse, $next) {
        //Get the session instance
        $session = AuraSession::getSession($request);
    }
]);
```

### BasePath

Strip off the prefix from the uri path of the request. This is useful to combine with routers if the root of the website is in a subdirectory. For example, if the root of your website is `/web/public`, a request with the uri `/web/public/post/34` will be converted to `/post/34`.

```php
use Psr7Middlewares\Middleware;

$dispatcher = $relay->getInstance([
    Middleware::BasePath('/web/public'),
]);
```

### BasicAuthentication

Implements the [basic http authentication](http://php.net/manual/en/features.http-auth.php). You have to pass an array with all users and password allowed:

```php
use Psr7Middlewares\Middleware;

$dispatcher = $relay->getInstance([

    Middleware::BasicAuthentication()
        ->users([
            'username1' => 'password1',
            'username2' => 'password2'
        ])
        ->realm('My realm') //(optional) change the realm value
]);
```

### Cache

To save and reuse responses based in the Cache-Control: max-age directive and Expires header.

```php
use Psr7Middlewares\Middleware;

$dispatcher = $relay->getInstance([

    Middleware::Cache()
        ->storage('cache/responses') //the directory to store the cache

    function($request, $response, $next) {
        //Cache the response 1 hour
        return $response->withHeader('Cache-Control', 'max-age=3600');
    }
]);
```

### ClientIp

Detects the client ip(s). You can configure the allowed headers.

```php
use Psr7Middlewares\Middleware;
use Psr7Middlewares\Middleware\ClientIp;

$dispatcher = $relay->getInstance([

    Middleware::ClientIp()
        ->headers([
            'Client-Ip',
            'X-Forwarded-For',
            'X-Forwarded'
        ]), //(optional) to change the trusted headers

    function ($request, $response, $next) {
        //Get the user ip
        $ip = ClientIp::getIp($request);

        //Get all ips found in the headers
        $all_ips = array_implode(', ', ClientIp::getIps($request));

        return $next($request, $response);
    }
]);
```

### DigestAuthentication

Implements the [digest http authentication](http://php.net/manual/en/features.http-auth.php). You have to pass an array with all users and password allowed as first argument:

```php
use Psr7Middlewares\Middleware;

$dispatcher = $relay->getInstance([

    Middleware::DigestAuthentication()
        ->users([
            'username1' => 'password1',
            'username2' => 'password2'
        ])
        ->realm('My realm') //(optional) custom realm value
        ->nonce(uniqid()) //(optional) custom nonce value
]);
```

### ErrorHandler

Executes a handler if the response returned by the next middlewares has any error (status code 400-599). You can catch also the exceptions throwed. To use it with an error handler like [whoops](https://github.com/filp/whoops), you can register it to catch exceptions and other errors using the methods `before` and `after`. These methods receives a callable wrapping your handler that returns a ResponseInterface object.

```php
use Psr7Middlewares\Middleware;
use Psr7Middlewares\Middleware\ErrorHandler;

function errorHandler($request, $response, $myApp) {
    switch ($response->getStatusCode()) {
        case 404:
            return 'Page not found';

        case 500:
            //you can get the exception catched
            $exception = ErrorHandler::getException($request);

            return 'Server error: '.$exception->getMessage();

        default:
            return 'There was an error'
    }
}

$whoops = new Whoops\Run();

$dispatcher = $relay->getInstance([

    Middleware::ErrorHandler()
        //My error handler
        ->handler('errorHandler')

        //(optional) append arguments to the handler
        ->arguments($myApp)

        //(optional) register your function to catch exceptions or other errors
        ->before(function ($handler) use ($whoops) {
           $whoops->pushHandler(function () use ($handler) {
                echo $handler()->getBody();
           });
        })

        //(optional) unregister the error catcher
        ->after(function ($handler) use ($whoops) {
            $whoops->popHandler();
        })

        //(optional) catch exceptions, if you don't use an external library for that
        ->catchExceptions(true)
]);
```

### FastRoute
To use [FastRoute](https://github.com/nikic/FastRoute) as a middleware.

```php
use Psr7Middlewares\Middleware;

$router = FastRoute\simpleDispatcher(function (FastRoute\RouteCollector $r) {

    $r->addRoute('GET', '/blog/{id:[0-9]+}', function ($request, $response, $app) {
        return 'This is the post number'.$request->getAttribute('id');
    });
});

$dispatcher = $relay->getInstance([

    Middleware::FastRoute()
        ->router($router) //Instance of FastRoute\Dispatcher
        ->argument($myApp) //(optional) arguments appended to the controller
]);
```

### Firewall

Uses [M6Web/Firewall](https://github.com/M6Web/Firewall) to provide a IP filtering. This middleware deppends of **ClientIp** (to extract the ips from the headers).

[See the ip formats allowed](https://github.com/M6Web/Firewall#entries-formats) for trusted/untrusted options:

```php
use Psr7Middlewares\Middleware;

$dispatcher = $relay->getInstance([

    //needed to capture the user ips
    Middleware::ClientIp(),

    //set the firewall
    Middleware::Firewall()
        ->trusted(['123.0.0.*']) //(optional) ips allowed
        ->untrusted(['123.0.0.1']) //(optional) ips not allowed
]);
```

### FormatNegotiation

Uses [willdurand/Negotiation](https://github.com/willdurand/Negotiation) to detect and negotiate the format of the document using the url extension and/or the `Accept` http header. It also adds the `Content-Type` header to the response if it's missing.

```php
use Psr7Middlewares\Middleware;
use Psr7Middlewares\Middleware\FormatNegotiation;

$dispatcher = $relay->getInstance([

    Middleware::FormatNegotiation()
        ->negotiator($negotiator) //(optional) Instance of Negotiation\FormatNegotiator
        ->addFormat('pdf', ['application/pdf', 'application/x-download']) //(optional) add new formats and mimetypes
    },

    function ($request, $response, $next) {
        //get the format (for example: html)
        $format = FormatNegotiation::getFormat($request);

        return $next($request, $response);
    }
]);
```

### LanguageNegotiation

Uses [willdurand/Negotiation](https://github.com/willdurand/Negotiation) to detect and negotiate the client language. You must provide an array with all available languages:

```php
use Psr7Middlewares\Middleware;
use Psr7Middlewares\Middleware\LanguageNegotiation;

$dispatcher = $relay->getInstance([

    Middleware::LanguageNegotiation()
        ->languages(['gl', 'en', 'es']), //Available languages

    function ($request, $response, $next) {
        //Get the preferred language
        $language = LanguageNegotiation::getLanguage($request);

        return $next($request, $response);
    }
]);
```

### Minify

Uses [mrclay/minify](https://github.com/mrclay/minify) to minify the html, css and js code from the responses.

```php
use Psr7Middlewares\Middleware;

$dispatcher = $relay->getInstance([
    
    Middleware::Minify()
        ->forCache(true) //(optional) only minify cacheable responses (see SaveResponse)
        ->inlineCss(false) //(optional) enable/disable inline css minification
        ->inlineJs(false) //(optional) enable/disable inline js minification
]);
```

### SaveResponse

Saves the response content into a file if all of the following conditions are met:

* The method is `GET`
* The status code is `200`
* The `Cache-Control` header does not contain `no-cache` value
* The request has not query parameters.

This is useful for cache purposes

```php
use Psr7Middlewares\Middleware;

$dispatcher = $relay->getInstance([

    Middleware::SaveResponse()
        ->storage('path/to/document/root') //Path where save the responses
        ->basePath('public') //(optional) basepath ignored from the request uri
]);
```

### TrailingSlash

Removes the trailing slash of the path. For example, `/post/23/` will be converted to `/post/23`. If the path is `/` it won't be converted. Useful if you have problems with the router.

```php
use Psr7Middlewares\Middleware;

$dispatcher = $relay->getInstance([

    Middleware::TrailingSlash()
        ->addSlash(true) //(optional) to add the trailing slash instead remove
        ->basePath('public') //(optional) basepath
]);
```

## Contribution

New middlewares are appreciated. Just create a pull request.
