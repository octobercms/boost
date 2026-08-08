# October CMS

This application uses **October CMS**, a Laravel-based content management system with its own conventions and architecture. October CMS patterns take precedence over standard Laravel patterns.

## Critical Differences from Laravel

- **Do not suggest** Livewire, Inertia.js, Blade components, or Laravel Folio - October CMS has its own frontend architecture.
- **Do not suggest** Laravel form requests for validation - October uses model-based validation via the `Validation` trait.
- **Do not suggest** Laravel controllers with route model binding - October uses backend controllers with behaviors.
- **Do not suggest** `resources/views/` Blade templates - October uses Twig-based CMS themes in the `themes/` directory and PHP-based partials in `controllers/` and `models/` directories.
- **Do not use** `php artisan make:model` or `php artisan make:controller` - October has its own scaffolding commands: `php artisan create:plugin`, `php artisan create:model`, `php artisan create:controller`, `php artisan create:component`.

## Architecture Overview

October CMS is built on these pillars:

- **Plugins** - modular packages in `plugins/{author}/{name}/` that extend the CMS. Each has a `Plugin.php` registration file extending `PluginBase`.
- **Themes** - file-based frontend templates in `themes/{name}/` using Twig markup with pages, layouts, partials, and content files.
- **Backend** - admin panel powered by controller behaviors (FormController, ListController, RelationController) with YAML-driven configuration.
- **Tailor** - headless CMS feature using YAML blueprints to define content structures without writing code.
- **AJAX Framework** - built-in AJAX system using `data-request` attributes or the `jax` JavaScript API to call server-side handlers.
- **Vue Components** - opt-in Vue 3 rendering for CMS components, using the `{% framework vue %}` and `{% vuecomponents %}` Twig tags with the `oc.createVueApp` / `oc.mountVueApp` factory.

## Plugin Structure

All custom code lives in plugins. A typical plugin structure:

```
plugins/acme/blog/
├── Plugin.php              ← Registration file
├── controllers/
│   └── Posts.php           ← Backend controller
│       └── posts/          ← Controller views directory
│           ├── config_list.yaml
│           ├── config_form.yaml
│           ├── _list_toolbar.php
│           ├── index.php
│           ├── create.php
│           └── update.php
├── models/
│   └── Post.php            ← Eloquent model
│       └── post/           ← Model config directory
│           ├── fields.yaml
│           └── columns.yaml
├── components/
│   └── BlogPost.php        ← CMS component
├── updates/
│   ├── version.yaml        ← Version history
│   └── create_posts_table.php
└── lang/
    └── en/
        └── lang.php
```

## Model Conventions

October CMS models extend `Model` (aliased from `October\Rain\Database\Model`) and use **array-based relationship definitions** instead of Laravel's fluent methods:

```php
class Post extends Model
{
    use \October\Rain\Database\Traits\Validation;

    protected $table = 'acme_blog_posts';

    public $rules = [
        'title' => 'required',
        'slug' => 'required|unique:acme_blog_posts',
    ];

    protected $jsonable = ['metadata'];

    public $belongsTo = [
        'category' => \Acme\Blog\Models\Category::class,
    ];

    public $hasMany = [
        'comments' => [\Acme\Blog\Models\Comment::class, 'delete' => true],
    ];

    public $attachOne = [
        'featured_image' => \System\Models\File::class,
    ];

    public $attachMany = [
        'gallery' => \System\Models\File::class,
    ];
}
```

Key differences from Laravel models:
- Relationships are defined as **public array properties** (`$hasOne`, `$hasMany`, `$belongsTo`, `$belongsToMany`, `$morphTo`, `$morphOne`, `$morphMany`, `$morphToMany`, `$morphedByMany`, `$attachOne`, `$attachMany`), not fluent methods.
- Validation is handled by the `Validation` trait with `$rules`, `$customMessages`, and `$attributeNames` properties.
- File attachments use `$attachOne` and `$attachMany` with `System\Models\File`.
- JSON columns use the `$jsonable` property (not `$casts`).
- Table names follow the pattern `{author}_{plugin}_{plural_name}` (e.g., `acme_blog_posts`).
- Model events use method overrides (`beforeCreate`, `afterSave`, etc.) not closures.

## Backend Controllers

Backend controllers use behaviors defined in YAML configuration files:

```php
class Posts extends \Backend\Classes\Controller
{
    public $implement = [
        \Backend\Behaviors\FormController::class,
        \Backend\Behaviors\ListController::class,
    ];

    public $formConfig = 'config_form.yaml';
    public $listConfig = 'config_list.yaml';
}
```

## AJAX Framework

Use `data-request` attributes to call server-side handlers:

```html
<form data-request="onSubmit" data-request-update="{ result: '#resultDiv' }">
    <input name="name" />
    <button type="submit">Send</button>
</form>
```

Handlers are PHP functions prefixed with `on`:

```php
function onSubmit()
{
    $this['result'] = input('name');
}
```

## Event System

October CMS uses a global event system for extensibility. Events are fired with `Event::fire()` and listened to with `Event::listen()`. Register listeners in Plugin `boot()` method:

```php
public function boot()
{
    \Event::listen('backend.form.extendFields', function ($widget) {
        // Extend form fields
    });
}
```

Common event patterns:
- `backend.form.extendFields` - extend backend forms
- `backend.list.extendColumns` - extend backend lists
- `backend.filter.extendScopes` - extend list filters
- `model.beforeSave` / `model.afterSave` - model lifecycle (use local events via `$model->bindEvent()`)
- Use `Event::fire('acme.blog.eventName', [$arg1])` for custom events

Models also support local events via the Emitter trait:

```php
$model->bindEvent('model.afterSave', function () use ($model) {
    // Respond to save
});
```

## Vue Components

October provides a Vue 3 component framework shared by the backend panel and the CMS. A component is three files: a PHP class, an ESM JavaScript module, and a template partial.

### Authoring the class

Scaffold with `create:vuecomponent Acme.Blog PostEditor`. The class extends `System\Classes\VueComponentBase` and lives in `vuecomponents/`:

```php
namespace Acme\Blog\VueComponents;

use System\Classes\VueComponentBase;

class PostEditor extends VueComponentBase
{
    /**
     * @var string componentName is the Vue component tag name (kebab-case).
     */
    protected $componentName = 'acme-blog-post-editor';

    /**
     * @var array require lists dependent Vue component classes.
     */
    protected $require = [
        \Backend\VueComponents\Modal::class,
    ];
}
```

The generated files follow a fixed layout, derived from the lowercased class name:

```
vuecomponents/
├── PostEditor.php                      ← Component class
└── posteditor/
    ├── assets/js/posteditor.js         ← ESM module (export default { ... })
    ├── assets/css/posteditor.css       ← Optional, auto-loaded if present
    └── partials/_posteditor.php        ← Template partial
```

- `$componentName` is the kebab-case HTML tag; namespace it to avoid collisions.
- `$require` lists dependency component classes, registered automatically before this one.
- Override `prepareVars()` to pass PHP data into the partial, and `registerSubcomponents()` to add nested components (each subcomponent needs its own `.js` and `_partial.php`).
- The ESM module exports a Vue Options-API definition; the template is the partial, not an inline `template` string.

### Registering the component

Backend controllers register in the constructor or an action; CMS components register in `init()`:

```php
class MyComponent extends ComponentBase
{
    public function init()
    {
        $this->registerVueComponent(\Acme\Blog\VueComponents\PostEditor::class);
    }
}
```

Registering in `init()` makes the component available for both page renders and AJAX requests.

### Frontend wiring (CMS only)

The backend layout outputs registered components automatically. On the frontend, the theme opts in with two Twig tags:

```twig
{% framework vue %}

<div id="app">
    <acme-blog-post-viewer :post-id="7"></acme-blog-post-viewer>
</div>

{% vuecomponents %}

<script type="module">
    oc.mountVueApp('#app');
</script>
```

- `{% framework vue %}` ships the Vue library only, exposed as the `window.Vue` global. Omit it to bring your own Vue (bundle or CDN) exposed as `window.Vue`.
- `{% vuecomponents %}` ships the client factory (`oc.createVueApp` / `oc.mountVueApp`) and the registered component templates. Place it before your mounting script.
- Frontend component ESM files must read the `Vue` global (no importmap on the frontend), not `import ... from 'vue'`.
- AJAX partials that register components are wired automatically; the page must already carry the library and `{% vuecomponents %}`.

## Settings Models

Plugin settings use `SettingModel` - not custom database tables:

```php
class Settings extends \System\Models\SettingModel
{
    public $settingsCode = 'acme_blog_settings';
    public $settingsFields = 'fields.yaml';
}
```

Read/write: `Settings::get('key')`, `Settings::set('key', 'value')`.

## Artisan Commands

### Scaffolding

Command | Description
--- | ---
`create:plugin Acme.Blog` | New plugin with registration file
`create:model Acme.Blog Post` | Model with migration and YAML configs
`create:controller Acme.Blog Posts` | Backend controller with views
`create:component Acme.Blog BlogPost` | CMS component
`create:vuecomponent Acme.Blog PostEditor` | Vue 3 component (backend or CMS)
`create:command Acme.Blog MyCommand` | Console command
`create:migration Acme.Blog AddStatusColumn` | Migration file
`create:formwidget Acme.Blog MyWidget` | Custom form widget
`create:filterwidget Acme.Blog MyFilter` | Custom filter widget
`create:reportwidget Acme.Blog MyReport` | Dashboard report widget
`create:contentfield Acme.Blog MyField` | Tailor content field
`create:job Acme.Blog ProcessData` | Queue job class
`create:factory Acme.Blog PostFactory` | Model factory
`create:seeder Acme.Blog PostSeeder` | Database seeder
`create:test Acme.Blog PostTest` | Test class

### System

- `php artisan october:migrate` - run all plugin migrations
- `php artisan october:fresh` - delete the demo theme and start fresh
- `php artisan plugin:refresh Acme.Blog` - refresh a plugin's migrations

### Testing

- `php artisan plugin:test Acme.Blog` - run all tests for a plugin
- `php artisan plugin:test Acme.Blog --filter=PostTest` - run a specific test class
- `php artisan plugin:test Acme.Blog --filter=testCreatePost` - run a specific test method

## Running Plugin Tests

- **Always use `php artisan plugin:test`** to run plugin tests. Do not use `php vendor/bin/phpunit` directly — the root `phpunit.xml` overrides `PLUGINS_PATH` to a fixtures directory, which prevents plugin dependencies from being found and migrated.
- The `plugin:test` command automatically locates the plugin's own `phpunit.xml` and passes it as the `--configuration` to PHPUnit.
- All PHPUnit options can be passed through: `--filter`, `--stop-on-failure`, `--group`, etc.
- Plugin tests extend `PluginTestCase` which auto-migrates core modules, the current plugin, and all its `$require` dependencies into an in-memory SQLite database.

## Conventions

- Check sibling files for existing patterns before writing new code.
- Follow the naming conventions: `Author\Plugin` namespace, snake_case table names, StudlyCase class names.
- Use `~/plugins/acme/blog/models/post/fields.yaml` path notation with `~` prefix for absolute plugin paths in YAML configs.
- Use `$/acme/blog/models/post/fields.yaml` path notation with `$` prefix as an alternative absolute path syntax.
