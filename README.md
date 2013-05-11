Simple Nette REST API
=====================
This repository is being developed.

Do not use it on production. It is only for study purposes.

### Content
- [Requirements](#requirements)
- [Installation & setup](#installation--setup)
- [Neon configuration](#neon-configuration)
- [Sample usage](#sample-usage)
- [Simple CRUD resources](#simple-crud-resources)
- [Accessing input data](#accessing-input-data)
- [Security & authentication](#security--authentication)

Requirements
------------
Drahak/Restful requires PHP version 5.3.0 or higher. The only production dependency is [Nette framework 2.0.x](http://www.nette.org).

Installation & setup
--------------------
The easist way is to use [Composer](http://doc.nette.org/en/composer)

	$ composer require drahak/restful:@dev

Then add following code to you app bootstrap file before creating container:

```php
Drahak\Restful\DI\Extension::install($configurator);
```

Neon configuration
------------------
You can configure Drahak\Restful library in config.neon in section `restful`:

```yaml
restful:
	cacheDir: '%tempDir%/cache'
	routes:
		prefix: resources
		module: 'RestApi'
		autoGenerated: TRUE
		panel: TRUE
```

- `cacheDir`: not much to say, just directory where to store cache
- `routes.prefix`: mask prefix to resource routes (only for auto generated routes)
- `routes.module`: default module to resource routes (only for auto generated routes)
- `routes.autoGenerated`: if `TRUE` the library auto generate resource routes from Presenter action method annotations (see below)
- `routes.panel`: if `TRUE` the resource routes panel will appear in your nette debug bar

#### Resource routes panel
It is enabled by default but you can disable it by setting `restful.routes.panel` to `FALSE`. This panel show you all REST API resources routes (exactly all routes in default route list which implements `IResourceRouter` interface). This is useful e.g. for developers who develop client application, so they have all API resource routes in one place.
![REST API resource routes panel](http://files.drahak.eu/restful-routes-panel.png "REST API resource routes panel")

Sample usage
------------

Create `BasePresenter`:

```php
<?php
namespace ResourcesModule;

use Drahak\Restful\Application\ResourcePresenter;
use Drahak\Restful\IResource;

/**
 * BasePresenter
 * @package ResourcesModule
 * @author Drahomír Hanák
 */
abstract class BasePresenter extends ResourcePresenter
{

    /** @var string */
    protected $defaultMimeType = IResource::JSON;

}
```

Then create your API resource presenter:

```php
<?php
namespace ResourcesModule;

use Drahak\Restful\IResource;

/**
 * SamplePresenter resource
 * @package ResourcesModule
 * @author Drahomír Hanák
 */
class SamplePresenter extends BasePresenter
{

   protected $typeMap = array(
       'json' => IResource::JSON,
       'xml' => IResource::XML
   );

   /**
    * @GET sample[.<type xml|json>]
    */
   public function actionContent($type = 'json')
   {
       $this->resource->title = 'REST API';
       $this->resource->subtitle = '';
       $this->sendResource($this->typeMap[$type]);
   }

   /**
    * @GET sample/detail
    */
   public function actionDetail()
   {
       $this->resource->message = 'Hello world';
   }

}
```

See `@GET` annotation. There are also available annotations `@POST`, `@PUT`, `@HEAD`, `@DELETE`. This allows Drahak\Restful library to generate API routes for you so you don't need to do it manualy. But it's not neccessary! You can define your routes using `IResourceRoute` or its default implementation such as:

```php
<?php
use Drahak\Restful\Application\Routes\ResourceRoute;

$anyRouteList[] = new ResourceRoute('sample[.<type xml|json>]', 'Resources:Sample:content', ResourceRoute::GET);
```

There is only one more parameter unlike the Nette default Route, the request method. This allows you to generate same URL for e.g. GET and POST method. You can pass this parameter to route as a flag so you can combine more request methods such as `ResourceRoute::GET | ResourceRoute::POST` to listen on GET and POST request method in the same route.

You can also define action names dictionary for each reqest method:

```php
<?php
new ResourceRoute('myResourceName', array(
    'presenter' => 'MyResourcePresenter',
    'action' => array(
        ResourceRoute::GET => 'content',
        ResourceRoute::DELETE => 'delete'
    )
), ResourceRoute::GET | ResourceRoute::DELETE);
```

Simple CRUD resources
---------------------
Well it's nice but in many cases I define only CRUD operations so how can I do it more intuitively? Use `CrudRoute`! This childe of `ResourceRoute` predefines base CRUD operations for you. Namely, it is `Presenter:create` for POST method, `Presenter:read` for GET, `Presenter:update` for PUT and `Presenter:delete` for DELETE. Then your router will look like this:

```php
<?php
new CrudRoute('<module>/crud', 'MyResourcePresenter');
```
Note the second parameter, metadata. You can define only Presenter not action name. This is because the action name will be replaced by value from actionDictionary (`[CrudRoute::POST => 'create', CrudRoute::GET => 'read', CrudRoute::PUT => 'update', CrudRoute::DELETE => 'delete']`) which is property of `ResourceRoute` so even of `CrudRoute` since it is its child. Also note that we don't have to set flags. Default flags are setted to `CrudRoute::CRUD` so the route will match all request methods.

Then you can simple define your CRUD resource presenter:

```php
<?php
namespace ResourcesModule;

/**
 * CRUD resource presenter
 * @package ResourcesModule
 * @author Drahomír Hanák
 */
class CrudPresenter extends BasePresenter
{

    public function actionCreate()
    {
        $this->resource->action = 'Create';
    }

    public function actionRead()
    {
        $this->resource->action = 'Read';
    }

    public function actionUpdate()
    {
        $this->resource->action = 'Update';
    }

    public function actionDelete()
    {
        $this->resource->action = 'Delete';
    }

}
```

Accessing input data
--------------------
If you want to build REST API, you may also want to access query input data for all request methods (GET, POST, PUT, DELETE and HEAD). So the library defines input parser, which reads data and parse it to an array. Data are fetched from query string or from request body and parsed by `IMapper`. Default mapper is `QueryMapper` but you can set e.g. `XmlMapper` to parse request body as XML.

```php
<?php
namespace ResourcesModule;

/**
 * Sample resource
 * @package ResourcesModule
 * @author Drahomír Hanák
 */
class SamplePresenter extends BasePresenter
{

	/**
	 * @PUT <module>/sample
	 */
	public function actionUpdate()
	{
		$this->resource->message = isset($this->input->message) ? $this->input->message : 'no message';
	}

}
```
You can set any mapper to `$presenter->input` in Presenter or to `restful.input` by calling service method `setMapper(IMapper)`. Then you can send `PUT` request to `resources/sample` with e.g. JSON string in body: `{"message": "hello"}`. The library will choose correct request method and parse it with `$presenter->mapper`.

Good thing about it is that you don't care of request method. Nette Drahak REST API library will choose correct Input parser for you but it's still up to you, how to handle it. There is available `InputIterator` so you can iterate through input in presenter or use it in your own input parser as iterator.

So that's it. Enjoy and hope you like it!

Security & authentication
-------------------------
The library provides base support of secure API calls. It's based on sending hashed data with private key. Authentication process is as follows:

### Understanding authentication process
- Client: append request timestamp to request body.
- Client: hash all data with `hash_hmac` (sha256 algorithm) and with private key. Then append generated hash to request as `X-HTTP-AUTH-TOKEN` header (by default).
- Client: sends request to server.
- Server: accept client's request and calculate hash in the same way as client (using abstract template class `AuthenticationProcess`)
- Server: compares client's hash with hash that it generated in previous step.
- Server: also checks request timestamp and make difference. If it's bigger then 300 (5 minutes) throws exception. (this avoid something called [Replay Attack](http://en.wikipedia.org/wiki/Replay_attack))
- Server: catches any `SecurityException` that throws `AuthenticationProcess` and provides error response (in production, in development just throws exception)

Default `AuthenticationProcess` is `NullAuthentication` so all requests are unsecured. You can turn on authentication by setting `SecuredAuthentication` to `restful.authenticationProcess` or to `$presenter->authenticationProcess`.

___

TODO list
---------
What I plan to (sometime) implement.

- Better API resources panel. There are to many records in the table.
- `@resource` annotation (presenter class) to auto generate CrudRoute
- Refactor `RouteListFactory` for easier scalability
