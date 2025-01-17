# Plugin Registration

- [Introduction](#introduction)
    - [Directory structure](#directory-structure)
    - [Plugin namespaces](#namespaces)
- [Registration file](#registration-file)
    - [Supported methods](#registration-methods)
    - [Basic plugin information](#basic-plugin-information)
- [Routing and initialization](#routing-initialization)
- [Dependency definitions](#dependency-definitions)
- [Extending Twig](#extending-twig)
- [Navigation menus](#navigation-menus)
- [Registering middleware](#registering-middleware)
- [Elevated permissions](#elevated-plugin)
- [Plugin replacement & forking](#plugin-replace)
  - [Registering as a replacement](#plugin-replace-registration)
    - [Version constraints](#version-constraints)
  - [Aliases](#aliases)
    - [Config Aliases](#aliases-config)
    - [Lang Aliases](#aliases-lang)
    - [Settings Aliases](#aliases-settings)
    - [Navigation Aliases](#aliases-navigation)
  - [Handling migrations, seeders and table references](#plugin-replace-data)

<a name="introduction"></a>
## Introduction

Plugins are the foundation for adding new features to the CMS by extending it. This article describes the component registration. The registration process allows plugins to declare their features such as [components](components) or back-end menus and pages. Some examples of what a plugin can do:

1. Define [components](components).
1. Define [user permissions](../backend/users).
1. Add [settings pages](settings#backend-pages), [menu items](#navigation-menus), [lists](../backend/lists) and [forms](../backend/forms).
1. Create [database table structures and seed data](updates).
1. Alter [functionality of the core or other plugins](../services/events).
1. Provide classes, [back-end controllers](../backend/controllers-ajax), views, assets, and other files.

<a name="directory-structure"></a>
### Directory structure

Plugins reside in the **/plugins** subdirectory of the application directory. An example of a plugin directory structure:

    plugins/
      acme/              <=== Author name
        blog/            <=== Plugin name
          assets/
          classes/
          components/
          controllers/
          models/
          updates/
          ...
          Plugin.php     <=== Plugin registration file

Not all plugin directories are required. The only required file is the **Plugin.php** described below. If your plugin provides only a single [component](components), your plugin directory could be much simpler, like this:

    plugins/
      acme/              <=== Author name
        blog/            <=== Plugin name
          components/
          Plugin.php     <=== Plugin registration file

Plugin assets like css and js files must reside under the assets directory:

    plugins/
      acme/
        blog/
          assets/        <=== Assets directory
            css/
              styles.css
            js/
              custom.js

> **Note:** if you are developing a plugin for the [Marketplace](/marketplace), the [updates/version.yaml](updates) file is required.

<a name="namespaces"></a>
### Plugin namespaces

Plugin namespaces are very important, especially if you are going to publish your plugins on the [Winter Marketplace](https://wintercms.com/marketplace). When you register as an author on the Marketplace you will be asked for the author code which should be used as a root namespace for all your plugins. You can specify the author code only once, when you register. The default author code offered by the Marketplace consists of the author first and last name: JohnSmith. The code cannot be changed after you register. All your plugin namespaces should be defined under the root namespace, for example `\JohnSmith\Blog`.

<a name="registration-file"></a>
## Registration file

The **Plugin.php** file, called the *Plugin registration file*, is an initialization script that declares a plugin's core functions and information. Registration files can provide the following:

1. Information about the plugin, its name, and author.
1. Registration methods for extending the CMS.

Registration scripts should use the plugin namespace. The registration script should define a class with the name `Plugin` that extends the `\System\Classes\PluginBase` class. The only required method of the plugin registration class is `pluginDetails`. An example Plugin registration file:

    namespace Acme\Blog;

    class Plugin extends \System\Classes\PluginBase
    {
        public function pluginDetails()
        {
            return [
                'name' => 'Blog Plugin',
                'description' => 'Provides some really cool blog features.',
                'author' => 'ACME Corporation',
                'icon' => 'icon-leaf'
            ];
        }

        public function registerComponents()
        {
            return [
                'Acme\Blog\Components\Post' => 'blogPost'
            ];
        }
    }

<a name="registration-methods"></a>
### Supported methods

The following methods are supported in the plugin registration class:

Method | Description
------------- | -------------
**pluginDetails()** | returns information about the plugin.
**register()** | register method, called when the plugin is first registered.
**boot()** | boot method, called right before the request route.
**registerComponents()** | registers any [front-end components](components#component-registration) used by this plugin.
**registerFormWidgets()** | registers any [back-end form widgets](../backend/widgets#form-widget-registration) supplied by this plugin.
**registerListColumnTypes()** | registers any [custom list column types](../backend/lists#custom-column-types) supplied by this plugin.
**registerMailLayouts()** | registers any [mail view layouts](../services/mail#mail-template-registration) supplied by this plugin.
**registerMailPartials()** | registers any [mail view partials](../services/mail#mail-template-registration) supplied by this plugin.
**registerMailTemplates()** | registers any [mail view templates](../services/mail#mail-template-registration) supplied by this plugin.
**registerMarkupTags()** | registers [additional markup tags](#extending-twig) that can be used in the CMS.
**registerNavigation()** | registers [back-end navigation menu items](#navigation-menus) for this plugin.
**registerPermissions()** | registers any [back-end permissions](../backend/users#permission-registration) used by this plugin.
**registerReportWidgets()** | registers any [back-end report widgets](../backend/widgets#report-widget-registration), including the dashboard widgets.
**registerSchedule()** | registers [scheduled tasks](../plugin/scheduling#defining-schedules) that are executed on a regular basis.
**registerSettings()** | registers any [back-end configuration links](settings#link-registration) used by this plugin.
**registerValidationRules()** | registers any [custom validators](../services/validation#custom-validation-rules) supplied by this plugin.

<a name="basic-plugin-information"></a>
### Basic plugin information

The `pluginDetails` is a required method of the plugin registration class. It should return an array containing the following keys:

Key | Description
------------- | -------------
**name** | the plugin name, required.
**description** | the plugin description, required.
**author** | the plugin author name, required.
**icon** | a name of the plugin icon. The full list of available icons can be found in the [UI documentation](../ui/icon). Any icon names provided by this font are valid, for example **icon-glass**, **icon-music**, optional.
**iconSvg** | an SVG icon to be used in place of the standard icon. The SVG icon should be a rectangle and can support colors, optional.
**homepage** | a link to the author's website address, optional.

<a name="routing-initialization"></a>
## Routing and initialization

Plugin registration files can contain two methods `boot` and `register`. With these methods you can do anything you like, like register routes or attach handlers to events.

The `register` method is called immediately when the plugin is registered. The `boot` method is called right before a request is routed. So if your actions rely on another plugin, you should use the boot method. For example, inside the `boot` method you can extend models:

    public function boot()
    {
        User::extend(function($model) {
            $model->hasOne['author'] = ['Acme\Blog\Models\Author'];
        });
    }

> **Note:** The `boot` and `register` methods are not called during the update process to protect the system from critical errors. To overcome this limitation use [elevated permissions](#elevated-plugin).

Plugins can also supply a file named **routes.php** that contain custom routing logic, as defined in the [router service](../services/router). For example:

    Route::group(['prefix' => 'api_acme_blog'], function() {

        Route::get('cleanup_posts', function(){ return Posts::cleanUp(); });

    });

<a name="dependency-definitions"></a>
## Dependency definitions

A plugin can depend upon other plugins by defining a `$require` property in the [Plugin registration file](#registration-file), the property should contain an array of plugin names that are considered requirements. A plugin that depends on the **Acme.User** plugin can declare this requirement in the following way:

```php
namespace Acme\Blog;

class Plugin extends \System\Classes\PluginBase
{
    /**
     * @var array Plugin dependencies
     */
    public $require = ['Acme.User'];

    // [...]
}
```

Dependency definitions will affect how the plugin operates and [how the update process applies updates](../plugin/updates#update-process). The installation process will attempt to install any dependencies automatically, however if a plugin is detected in the system without any of its dependencies it will be disabled to prevent system errors.

Dependency definitions can be complex but care should be taken to prevent circular references. The dependency graph should always be directed and a circular dependency is considered a design error.

<a name="extending-twig"></a>
## Extending Twig

Custom Twig filters and functions can be registered in the CMS with the `registerMarkupTags` method of the plugin registration class.

[Twig options](https://twig.symfony.com/doc/2.x/advanced.html#filters) are also able to be passed to change the behavior of the registered filters & functions by providing an array with an `'options'` element containing the options to be passed at time of registration where the callable value would be provided normally. If options are provided, then the callable handler for the filter / function being registered must either be present in a `'callable'` element or as the first element of the array.

>**IMPORTANT:** All custom Twig filters & functions registered via the `MarkupManager` (i.e. `registerMarkupTags()` will have the `is_safe` option set to `['html']` by default, which means that Twig's automatic escaping is disabled by default (effectively it's as if the `| raw` filter was always located after your filter or function's output) unless you provide the `is_safe` option during registration (`'options' => ['is_safe' => []]`).

The next example registers three Twig filters and three functions.

```php
public function registerMarkupTags()
{
    return [
        'filters' => [
            // A global function, i.e str_plural()
            'plural' => 'str_plural',

            // A local method, i.e $this->makeTextAllCaps()
            'uppercase' => [$this, 'makeTextAllCaps']

            // Any callable with custom options defined - first element method
            'userInputToEmojis' => ['input_to_emojis', 'options' => ['is_safe' => []]],
        ],
        'functions' => [
            // A static method call, i.e Form::open()
            'form_open' => ['Winter\Storm\Html\Form', 'open'],

            // Using an inline closure
            'helloWorld' => function() { return 'Hello World!'; }

            // Any callable with custom options defined - named 'callable' method
            'goodbyeWorld' => [
                'callable' => ['Acme\Plugin\Nuke', 'boom'],
                'options'  => ['needs_environment' => true],
            ],
        ],
    ];
}

public function makeTextAllCaps($text)
{
    return strtoupper($text);
}
```

The following Twig custom options are available:

| Option | Type | Default | Description |
| ------ | ---- | ------- | ----------- |
| `needs_environment` | boolean | `false` | if true provides the current `TwigEnvironment` as the first argument to the filter call |
| `needs_context` | boolean | `false` | if true provides the current `TwigContext` as the first argument (second if `needs_environment also set`) to the filter call |
| `is_safe` | array | `[]` | array of languages (usually `html` or `all` are valid values) that the output of the filter / function is safe to be used on without escaping |
| `pre_escape` | string | `''` | (only filters) will pre-escape the value before it is passed to your filter for the language that you set (usually `'html'`) |
| `preserves_safety` | array | `[]` | (only filters) array of languages (usually `html`) that the filter will preserve the safety setting of for previous filters in the chain. i.e. if the previous filter in the chain says that its safe and doesn't require escaping then neither will this one, but if it says that it's unsafe and requires escaping then so will this one. |
| `is_variadic` | boolean | `false` | if true will pass any extra arguments provided to the filter as a single array as the last argument to the filter call |
| `deprecated` | boolean | `false` | if true marks the current filter as being deprecated (usually used with `alternative` to provide an alternative option |
| `alternative` | string | `''` | if `deprecated` is true, provides a recommended alternative filter to use instead. |


<a name="navigation-menus"></a>
## Navigation menus

Plugins can extend the back-end navigation menus by overriding the `registerNavigation` method of the [Plugin registration class](#registration-file). This section shows you how to add menu items to the back-end navigation area. An example of registering a top-level navigation menu item with two sub-menu items:

```php
public function registerNavigation()
{
    return [
        'blog' => [
            'label'       => 'Blog',
            'url'         => Backend::url('acme/blog/posts'),
            'icon'        => 'icon-pencil',
            'permissions' => ['acme.blog.*'],
            'order'       => 500,
            // Set counter to false to prevent the default behaviour of the main menu counter being a sum of
            // its side menu counters
            'counter'     => ['\Author\Plugin\Classes\MyMenuCounterService', 'getBlogMenuCount'],
            'counterLabel'=> 'Label describing a dynamic menu counter',
            // Optionally you can set a badge value instead of a counter to display a string instead of a numerical counter
            'badge'       => 'New'

            'sideMenu' => [
                'posts' => [
                    'label'       => 'Posts',
                    'icon'        => 'icon-copy',
                    'url'         => Backend::url('acme/blog/posts'),
                    'permissions' => ['acme.blog.access_posts'],
                    'counter'     => 2,
                    'counterLabel'=> 'Label describing a static menu counter',
                ],
                'categories' => [
                    'label'       => 'Categories',
                    'icon'        => 'icon-copy',
                    'url'         => Backend::url('acme/blog/categories'),
                    'permissions' => ['acme.blog.access_categories'],
                ]
            ]
        ]
    ];
}
```

When you register the back-end navigation you can use [localization strings](localization) for the `label` values. Back-end navigation can also be controlled by the `permissions` values and correspond to defined [back-end user permissions](../backend/users). The order in which the back-end navigation appears on the overall navigation menu items, is controlled by the `order` value. Higher numbers mean that the item will appear later on in the order of menu items while lower numbers mean that it will appear earlier on.

To make the sub-menu items visible, you may [set the navigation context](../backend/controllers-ajax#navigation-context) in the back-end controller using the `BackendMenu::setContext` method. This will make the parent menu item active and display the children in the side menu.

Key | Description
------------- | -------------
**label** | specifies the menu label localization string key, required.
**icon** | an icon name from the [Winter CMS icon collection](../ui/icon), optional.
**iconSvg** | an SVG icon to be used in place of the standard icon, the SVG icon should be a rectangle and can support colors, optional.
**url** | the URL the menu item should point to (ex. `Backend::url('author/plugin/controller/action')`, required.
**counter** | a numeric value to output near the menu icon. The value should be a number or a callable returning a number, optional.
**counterLabel** | a string value to describe the numeric reference in counter, optional.
**badge** | a string value to output in place of the counter, the value should be a string and will override the badge property if set, optional.
**attributes** | an associative array of attributes and values to apply to the menu item, optional.
**permissions** | an array of permissions the backend user must have in order to view the menu item (Note: direct access of URLs still requires separate permission checks), optional.
**code** | a string value that acts as an unique identifier for that menu option. **NOTE**: This is a system generated value and should not be provided when registering the navigation items.
**owner** | a string value that specifies the menu items owner plugin or module in the format "Author.Plugin". **NOTE**: This is a system generated value and should not be provided when registering the navigation items.

<a name="registering-middleware"></a>
## Registering middleware

To register a custom middleware, you can apply it directly to a Backend controller in your plugin by using [Controller middleware](../backend/controllers-ajax#controller-middleware), or you can extend a Controller class by using the following method.

```php
public function boot()
{
    \Cms\Classes\CmsController::extend(function($controller) {
        $controller->middleware('Path\To\Custom\Middleware');
    });
}
```

Alternatively, you can push it directly into the Kernel via the following.

```
public function boot()
{
    // Add a new middleware to beginning of the stack.
    $this->app['Illuminate\Contracts\Http\Kernel']
        ->prependMiddleware('Path\To\Custom\Middleware');

    // Add a new middleware to end of the stack.
    $this->app['Illuminate\Contracts\Http\Kernel']
        ->pushMiddleware('Path\To\Custom\Middleware');
}
```

<a name="elevated-plugin"></a>
## Elevated permissions

By default plugins are restricted from accessing certain areas of the system. This is to prevent critical errors that may lock an administrator out from the back-end. When these areas are accessed without elevated permissions, the `boot` and `register` [initialization methods](#routing-initialization) for the plugin will not fire.

Request | Description
------------- | -------------
**/combine** | the asset combiner generator URL
**/backend/system/updates** | the site updates context
**/backend/system/install** | the installer path
**/backend/backend/auth** | the backend authentication path (login, logout)
**winter:up** | the CLI command that runs all pending migrations
**winter:update** | the CLI command that triggers the update process
**winter:env** | the CLI command that converts configuration files to environment variables in a `.env` file
**winter:version** | the CLI command that detects the version of Winter CMS that is installed

Define the `$elevated` property to grant elevated permissions for your plugin.

```php
/**
 * @var bool Plugin requires elevated permissions.
 */
public $elevated = true;
```

<a name="plugin-replace"></a>
## Plugin replacement & forking

Plugin replacement is a feature that allows you to create a plugin that replaces (or overrides) another plugin. This is useful when you're forking a plugin to add your own functionality but want to be able to seamlessly migrate from and act as a drop in replacement for the original plugin (i.e. retaining original data, fulfilling other plugin's dependencies on the original plugin, etc).

<a name="plugin-replace-registration"></a>
### Registering as a replacement

To enable the plugin replacement feature, specify the identifier for the plugin your plugin is replacing in your plugin details along with the version constraints that define what versions of the plugin are able to be replaced by your plugin.

```php
public function pluginDetails()
{
    return [
        'name'     => 'Acme Plugin',
        'replaces' => [
            'Acme.Original' => '>=5.0 <=6.0.4'
        ],
    ];
}
```

<a name="version-constraints"></a>
#### Version constraints

Version constraints allow you to restrict your plugin to only override currently installed plugins of specific versions. The above example showcases only replacing a plugin if the original plugin is any version between `5.0` and `6.0.4` inclusive. Most of the time the actual version constraint you'll use will be much simpler, a simple `<2.0` to indicate all versions immediately prior to the version you first release your replacement plugin as.

This means you don't have to worry about new versions of the original plugin having changes that may conflict with your changes to the plugin.

Version constraints are specified in the [same format that Composer uses](https://getcomposer.org/doc/articles/versions.md#writing-version-constraints). Some valid examples would be:

- `1.0`
- `>=1.0.3`
- `<2.0`
- `>=1.5.0 <2.0.0`
- `self.version`

By specifying a version, your plugin will check what version the original plugin is installed at and only if it's version matches the constraint will it disable the original and enable the replacement. If this match fails, then the replacement will be disabled and the original plugin will stay enabled.

<a name="aliases"></a>
### Aliases

>**NOTE:** This is for reference only. By registering as a plugin replacement using the above feature Winter already handles registering these aliases throughout the system for you.

<!-- TODO: Group the into their own docs -->

Aliasing is a feature of Winter that allows for backwards compatibility and support for inheriting replaced plugins:

- Configs
- Lang
- Settings
- Navigation

<a name="aliases-config"></a>
#### Config

Config supports 2 different types of aliasing: `registerNamespaceAlias` & `registerPackageFallback`.

##### registerNamespaceAlias

This method allows for redirection of the alias to the namespace while accessing config values.

```php
Config::registerNamespaceAlias('winter.replacement', 'winter.original');
```

For example, register the following config as `plugins/winter/replacement/config/config.php`:

```php
<?php

return [
    'foo' => 'bar'
];
```

The config will be accessible via the alias registered:

```php
config('winter.original::foo'); // returns bar
```

##### registerPackageFallback

This method allows falling back to an aliased global config (a config specified in `/config/acme/plugin/config.php`).

```php
Config::registerPackageFallback('winter.replacement', 'winter.original');
```

The logic to this is as follows:

- If `/config/winter/replacement/config.php` exists it will be registered under the `winter.replacement` namespace.
- If `/config/winter/replacement/config.php` does not exist, it will check `/config/winter/original/config.php` and if found,
  it will be registered under the `winter.replacement`.

<a name="aliases-lang"></a>
#### Lang

Allows for redirection of calls to the alias and returns values from the namespace.

```php
Lang::registerNamespaceAlias('winter.replacement', 'winter.original');
```

For example, register the following config as `plugins/winter/replacement/lang/en/lang.php`:

```php
<?php

return [
    'foo' => 'bar'
];
```

The lang will be accessible via the alias registered:

```php
Lang::get('winter.original::foo'); // returns bar
```

<a name="aliases-settings"></a>
#### Settings

There are 2 methods for registering settings aliases. Firstly the aliases can be registered prior to the `PluginManager` init via `lazyRegisterOwnerAlias`.

```php
SettingsManager::lazyRegisterOwnerAlias('Winter.Replacement', 'Winter.Original');
```

If the `PluginManager` has been loaded, then aliases can be registered via:

```php
SettingsManager::instance()->registerOwnerAlias('Winter.Replacement', 'Winter.Original');
```

<a name="aliases-navigation"></a>
#### Navigation

There are 2 methods for registering settings aliases. Firstly the aliases can be registered prior to the `PluginManager` init via `lazyRegisterOwnerAlias`.

```php
NavigationManager::lazyRegisterOwnerAlias('Winter.Replacement', 'Winter.Original');
```

If the `PluginManager` has been loaded, then aliases can be registered via:

```php
NavigationManager::instance()->registerOwnerAlias('Winter.Replacement', 'Winter.Original');
```

<a name="plugin-replace-data"></a>
### Migrations, seeders and table references

When forking a plugin and using the replace functionality, you will need to handle migratign the data from the original plugin to your replacing plugin via  migrations, seeders and the model classes. To do this we recommend the following:

- Create a migration to rename tables
- Update models to reference your new table
- Check migrations for any usage of models

#### Table renaming

An example migration could look something like this:

```php
<?php namespace Winter\Plugin\Updates;

use Schema;
use Winter\Storm\Database\Updates\Migration;

class RenameTables extends Migration
{
    const TABLES = [
        'example',
        'foo',
        'bar'
    ];

    public function up()
    {
        foreach (self::TABLES as $table) {
            $from = 'acme_plugin_' . $table;
            $to = 'winter_plugin_' . $table;

            if (Schema::hasTable($from) && !Schema::hasTable($to)) {
                Schema::rename($from, $to);
            }
        }
    }

    public function down()
    {
        foreach (self::TABLES as $table) {
            $from = 'winter_plugin_' . $table;
            $to = 'acme_plugin_' . $table;

            if (Schema::hasTable($from) && !Schema::hasTable($to)) {
                Schema::rename($from, $to);
            }
        }
    }
}
```

#### Migrations using models

If an old migration (i.e. any migration that runs before the migration that renames the tables) is using a model to populate data, it will be referencing the new table and that will cause issues while updating. The solution to this is dynamically renaming the table before inserting/modifying data:

```php
ExampleModel::extend(function ($model) {
    $model->setTable('acme_plugin_example');
});

// execute seeding code

ExampleModel::extend(function ($model) {
    $model->setTable('winter_plugin_example');
});
```

If the models use the `unique` validation rule, you should make sure that the rule is implemented without any modifiers (i.e. just `'slug' => 'unique'`, not `'slug' => 'unique:winter_plugin_table'` so that the call to `$model->setTable()` will also take effect in that validation logic.
