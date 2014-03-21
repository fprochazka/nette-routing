Nette Routing: two-ways URL conversion
======================================

Routing is a two-way conversion between URL and presenter action. *Two-way* means that we can both determine what presenter URL links to, but also vice versa: generate URL for given action.


SimpleRouter
------------

Desired URL format is set by a *router*. The most plain implementation of router is [SimpleRouter | api:Nette\Application\Routers\SimpleRouter]. It can be used when there's no need for a specific URL format, or when `mod_rewrite` (or alternatives) is not available.

Addresses look like this:

```
http://example.com/index.php?presenter=Product&action=detail&id=123
```

The first parameter of `SimpleRouter` constructor is a default presenter action, ie. action to be executed if we open for example `http://example.com/` without additional parameters.

```php
// defaults to presenter 'Homepage' and action 'default'
$router = new Nette\Application\Routers\SimpleRouter('Homepage:default');
```

The second constructor parameter is optional and is used to pass additional [#flags] (only `SimpleRouter::ONE_WAY` and `SimpleRouter::SECURED` are supported).


Route: for prettier URLs
------------------------

Human-friendly URLs (also more cool & prettier) are easier to remember and do help SEO((search engine optimalization)). Nette Framework keeps current trends in mind and fully meets developers' desires.

All requests must be handled by `index.php` file. This can be accomplished e.g. by using Apache module `mod_rewrite` (see [how to enable mod_rewrite |troubleshooting#how-to-enable-mod-rewrite]) or Nginx's try_files (available by default).

Class Route can create addresses in pretty much any format one can though of. Let's start with a simple example generating a pretty URL for action `Product:default` with `id = 123`:

```
http://example.com/product/detail/123
```

The following snippet creates a `Route` object, passing path mask as the first argument and specifying default action in the second argument.

```php
// action defaults to presenter Homepage and action default
$route = new Route('<presenter>/<action>[/<id>]', 'Homepage:default');
```

This route is usable by all presenters and actions. Accepts paths such as `/article/edit/10` or `/catalog/list`, because the `id` part is wrapped in square brackets, which marks it as [optional | #optional sequences].

Because other parameters (`presenter` and `action`) do have default values (`Homepage` and `default`), they are optional too. If their value is the same as the default one, they are skipped while URL is generated. Link to `Product:default` generates only `http://example.com/product` and link to `Homepage:default` generates only `http://example.com/`.


Path Mask
---------

The simplest path mask consists only of a **static URL** and a target presenter action.

```php
$route = new Route('products', 'Products:default');
```

Most real masks however contain some **parameters**. Parameters are enclosed in angle brackets (e.g. `<year>`) and are passed to target presenter.

```php
$route = new Route('history/<year>', 'History:view');
```

Mask can also contain traditional GET arguments (query after a question mark). Neither validation expressions nor more complex structures are supported in this part of path mask, but you can set what key belongs to which variable:

``` php
$route = new Route('<presenter>/<action> ? id=<productId> & cat=<categoryId>', ...);
```

The parameters before a question mark are called *path parameters* and the parameters after a question mark are called *query parameters*.


```comment
TERMINOLOGY ISSUE:
I've decided to use terminilogy by @Panda (http://edu.lynt.cz/course/download/?fileId=10). @DG uses "absolute path" for what I call here "relative to server document root" and "absolute URL" for what I call "absolute path". Any idea how to resolve this?
```

Mask can not only describe path relative to application document root (web root), but can as well contain path relative to server document root (starts with a single slash) or absolute path with domain (starts with a double slash).

```php
// relative to application document root (www directory)
$route = new Route('<presenter>/<action>', ...);

// relative to server document root
$route = new Route('/<presenter>/<action>', ...);

// absolute path including hostname
$route = new Route('//<subdomain>.example.com/<presenter>/<action>', ...);
```

Absolute path mask may utilize the following variables:
- `%tld%` = top level domain, e.g. `com` or `org`
- `%domain%` = second level domain, e.g. `example.com`
- `%basePath%`

```php
$route = new Route('//www.%domain%/%basePath%/<presenter>/<action>', ...);
```



Route Collection
----------------

Because we usually define more than one route, we wrap them into a [RouteList | api:Nette\Application\Routers\RouteList].

```php
use Nette\Application\Routers\RouteList,
	Nette\Application\Routers\Route;

$router = new RouteList();
$router[] = new Route('rss.xml', 'Feed:rss');
$router[] = new Route('article/<id>', 'Article:view');
$router[] = new Route('<presenter>/<action>[/<id>]', 'Homepage:default');
```

.[note]
Unlike other frameworks, Nette does not require routes to be named.

It's important in which order are the routes defined as they are tried from top to bottom. The rule of thumb here is that routes are declared from the **most specific at the top** to the **most vague at the bottom**. Keep in mind that huge amount of routes can negatively effect application speed, mostly when generating links. It's worth to keep routes as simple as possible.

.[note]
If no route is found, a [BadRequestException | api:Nette\Application\BadRequestException] is thrown, which is shown as 404 Not Found to the user.



Default Values
--------------

Each parameter may have defined a default value in the mask:

```php
$route = new Route('<presenter=Homepage>/<action=default>/<id=>');
```

Or utilizing an array:

```php
$route = new Route('<presenter>/<action>/<id>', array(
	'presenter' => 'Homepage',
	'action' => 'default',
	'id' => NULL,
));

// equals to the following complex notation
$route = new Route('<presenter>/<action>/<id>', array(
	'presenter' => array(
		Route::VALUE => 'Homepage',
	),
	'action' => array(
		Route::VALUE => 'default',
	),
	'id' => array(
		Route::VALUE => NULL,
	),
));
```

Default values for `<presenter>` and `<action>` can also be written as a string in the second constructor parameter.

```php
$route = new Route('<presenter>/<action>/<id=>', 'Homepage:default');
```



Validation Expressions
----------------------

Each parameter may have defined a **regular expression** which it needs to match. This regular expression is checked both when matching and generating URL. For example let's set `id` to be only numerical, using `\d+` regexp:

```php
// regexp can be defined directly in the path mask after parameter name
$route = new Route('<presenter>/<action>[/<id \d+>]', 'Homepage:default');

// equals to the following complex notation
$route = new Route('<presenter>/<action>[/<id>]', array(
	'presenter' => 'Homepage',
	'action' => 'default',
	'id' => array(
		Route::PATTERN => '\d+',
	),
));
```


Default validation expression for path parameters is `[^/]+`, meaning all characters but a slash. If a parameter is supposed to match a slash as well, we can set the regular expression to `.+`.

Regular expressions are *case-insensitive* by default. You need to set `Route::CASE_SENSITIVE` flag to make them *case-sensitive*.


Optional Sequences
------------------

Square brackets denote optional parts of mask. Any part of mask may be set as optional, including those containing parameters:

```php
$route = new Route('[<lang [a-z]{2}>/]<name>', 'Article:view');

// Accepted URLs:      Parameters:
//   /en/download        action => view, lang => en, name => download
//   /download           action => view, lang => NULL, name => download
```

Obviously, if a parameter is inside an optional sequence, it's optional too and defaults to `NULL`. Sequence should define it's surroundings, in this case a slash which must follow a parameter, if set.
The technique may be used for example for optional language subdomains:

```php
$route = new Route('//[<lang=en>.]%domain%/<presenter>/<action>', ...);
```

Sequences may be freely nested and combined:

```php
$route = new Route(
	'[<lang [a-z]{2}>[-<sublang>]/]<name>[/page-<page=0>]',
	'Homepage:default'
);

// Accepted URLs:
//   /cs/hello
//   /en-us/hello
//   /hello
//   /hello/page-12
```


URL generator tries to keep the URL as short as possible (while unique), so what can be omitted is not used. That's why `index[.html]`  route generates `/index`. This behavior can be inverted by writing an exclamation mark after the leftmost square bracket that denotes the respective optional sequence:

```php
// accepts both /hello and /hello.html, generates /hello
$route = new Route('<name>[.html]');

// accepts both /hello and /hello.html, generates /hello.html
$route = new Route('<name>[!.html]');
```


Optional parameters (ie. parameters having default value) without square brackets do behave as if wrapped like this:

```php
$route = new Route('<presenter=Homepage>/<action=default>/<id=>');

// equals to:
$route = new Route('[<presenter=Homepage>/[<action=default>/[<id>]]]');
```

If we would like to change how the rightmost slashes are generated, that is instead of `/homepage/` get a `/homepage`, we can adjust the route:

```php
$route = new Route('[<presenter=Homepage>[/<action=default>[/<id>]]]');
```


Filters and Translation
-----------------------

It's a good practice to write source code in English, but what if you need your application to run in a different environment? Simple routes such as:

```php
$route = new Route('<presenter>/<action>/<id>', array(
	'presenter' => 'Homepage',
	'action' => 'default',
	'id' => NULL,
));
```

will generate English URLs, such as `/product/detail/123`, `/cart` or `/catalog/view`. If we would like to translate those URLs, we can use a *dictionary* defined under `Route::FILTER_TABLE` key. We'd extend the route so:

```php
$route = new Route('<presenter>/<action>/<id>', array(
	'presenter' => array(
		Route::VALUE => 'Homepage', // default value
		Route::FILTER_TABLE => array(
			// translated string in URL => presenter
			'produkt' => 'Product',
			'einkaufswagen' => 'Cart',
			'katalog' => 'Catalog',
		),
	),
	'action' => array(
		Route::VALUE => 'default',
		Route::FILTER_TABLE => array(
			'sehen' => 'view',
		),
	),
	'id' => NULL,
));
```

Multiple keys under `Route::FILTER_TABLE` may have the same value. That's how aliases are created. The last value is the canonical one (used for generating links).

Dictionaries may be applied to any path parameter. If a translation is not found, the original (non-translated) value is used. The route by default accepts both translated (e.g. `/einkaufswagen`) and original (e.g. `/cart`) URLs. If you would like to accept only translated URLs, you need to add `Route::FILTER_STRICT => TRUE` to the route definition.


Besides setting dictionaries as arrays, it's possible to set **input and output filters**. Input and output filter are callbacks defined under `Route::FILTER_IN` and `Route::FILTER_OUT` keys respectively.

```php
$route = new Route('<presenter=Homepage>/<action=default>', array(
	'action' => array(
		Route::FILTER_IN => function($action) {
			return strrev($action);
		},
		Route::FILTER_OUT => function($action) {
			return strrev($action);
		},
	),
));
```

The **input filter** accepts value from URL and should return a value which will be passed to a presenter. The **output filter** accepts value from presenter and should return a value which will be used in URL. If any of those filters is unable to filter the value (usually because it is invalid), it should return `NULL`.


Besides filters for specific parameters, you can also define **global filters** which accepts an associative array with all parameters and returns an array with filtered parameters. Global filters are defined under `NULL` key.

```php
$route = new Route('<presenter>/<action>', array(
	'presenter' => 'Homepage',
	'action' => 'default',
	NULL => array(
		Route::FILTER_IN => function(array $params) {
			// ...
			return $params;
		},
		Route::FILTER_OUT => function(array $params) {
			// ...
			return $params;
		},
	),
));
```

You can use *global filters* to filter certain parameter based on a value of another parameter, e.g. translate `<presenter>` and `<action>` based on `<lang>`.



-----

[![Build Status](https://secure.travis-ci.org/nette/routing.png?branch=master)](http://travis-ci.org/nette/routing)
