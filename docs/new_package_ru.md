## How to make a package (plugin) in Orchid step by step. 
 
In this lesson, we will learn how to create plugins for Orchid, the difference between plugins and a project is that it can be easily connected to other projects. 

Our plugin will display [settings] (https://orchid.software/en/docs/settings) Orchid, and will also create the ability to edit them.

![](https://github.com/orchidcommunity/XSetting/blob/master/docs/imgs/create_plugin1.gif)

The settings are easily displayed in the blade template engine like this `{{setting ('phone')}}`.


### Directory and plugin connection.
First you need to create a directory where our projects will be contained.

1. In the root directory of the project, create a directory called package.

2. In the package directory, create a folder for our project. - For example XSetting.

In the composer.json file, add the paths to our project  
```
    "repositories": [ 
    { 
        "packagist.org": false, 
        "type": "path", 
        "url": "/home/youproject/package/xsetting" 
    }, 
```
Now, with the command `composer require" orchids / xsetting "`, the composer will also look for the project in our directory.
After completing these steps, you can also create a plugin not only for Orchid but also for any project under Laravel.
Of course, if you connect your plugin to packagist.org, then these steps are not needed.

### Creating a plugin.
Let's create the package structure

```
database/ migrations/2018_08_07_000000_create_options_for_settings_table.php
routes/route.php
src/Models/XSetting.php
src/Providers/XSettingProvider.php 
src/Http/Screens/XSettingList.php 
src/Http/Screens/XSettingEdit.php 
src/Http/Layouts/XSettingListLayout.php 
src/Http/Layouts/XSettingEditLayout.php 
Composer.json 
```

Let's start developing the extension.


#### 1. First, create a composer.json file for Composer
```
{
  "name": "orchids/xsetting",
  "description": "Extension setting package for Orchid Platform",
  "type": "library",
  "keywords": [
    "Orchid",
    "XSetting"
  ],
  "license": "MIT",
  "require": {
    "orchid/platform":"dev-master"
  },
  "autoload": {
    "psr-4": {
      "Orchids\\XSetting\\": "src/"
    }
  },
  "extra": {
    "laravel": {
      "providers": [
        "Orchids\\XSetting\\Providers\\XSettingProvider"
      ]
    }
  },
  "minimum-stability": "stable",
  "prefer-stable": true
}
```
Decoding of the main parameters
`" name ":" orchids / xsetting ",` - when executing a command in the `composer require" orchids / xsetting "` console, the composer will process this file.

`" require ": {" orchid / platform ":" dev-master "}` - the required dependencies to install the package.

`" psr-4 ": {" Orchids \\ XSetting \\ ":" src / "}` - classes whose path starts with `Orchids \ XSetting` will be associated with the` src / `directory in this package.

`" laravel ": {" providers ": [" Orchids \\ XSetting \\ Providers \\ XSettingProvider "]}` - After installation, it will launch the provider along the path `src / Providers / XSettingProvider.php`

#### 2. Now let's create a provider class that will add routing, the ability to give users access to this package, and add our plugin item to the menu.
Create a file `src / Providers / XSettingProvider.php`
```
namespace Orchids\XSetting\Providers;

use Illuminate\Console\Command;
use Illuminate\Support\ServiceProvider;
use Illuminate\Support\Facades\View;
use Orchid\Platform\Dashboard;

class XSettingProvider extends ServiceProvider
{
    public function boot(Dashboard $dashboard)
    {
        $this->dashboard = $dashboard;
        / *
        * Adding access permissions for users and roles.
        * /
        $this->app->booted(function () {
            $this->dashboard->registerPermissions(
                ItemPermission::group(__('Systems'))
                   ->addPermission('platform.systems.xsetting', __('Edit settings'))
            );
        });
		
        / *
        * Connecting the routing file
        * /
        $this->loadRoutesFrom(realpath(XSETTING_PATH.'/routes/route.php')); 
    
        / *
        * Connecting a class adding a menu item
        * /
        View::composer('platform::systems', MenuComposer::class); // To the system menu in the center
        //View::composer('platform::layouts.dashboard', MenuComposer::class);  // To the left menu
        
        /*
        * Including migration files to add them to the database, you need to execute `php artisan migrate`
        */
        $this->loadMigrationsFrom(realpath(XSETTING_PATH.'/database/migrations'));
    }
}
```

#### 3. First, let's create a routing file that will determine which screen is responsible for a particular route.
Let's create a routes / route.php file

```
use Orchids\XSetting\Http\Screens\XSettingEdit;
use Orchids\XSetting\Http\Screens\XSettingList;

Route::domain((string) config('platform.domain'))   // Loads the admin domain from the config
    ->prefix(Dashboard::prefix('/systems'))         // Loads the admin prefix from the config and adds / systems
    ->middleware(config('platform.middleware.private'))
    ->group(function (\Illuminate\Routing\Router $router, $path='platform.xsetting.') {
        $router->screen('xsetting/{xsetting}/edit', XSettingEdit::class)->name($path.'edit');
        $router->screen('xsetting/create', XSettingEdit::class)->name($path.'create');
        $router->screen('xsetting', XSettingList::class)->name($path.'list');
    });
```
Now, when generating the path `route ('platform.xsetting.list')`, the browser will generate approximately the following address `http: // yousite.name / dashboard / systems / xsetting`, and when you switch to it, run the file` src \ Http \ Screens \ XSettingList.php`

#### 4. Add an item to the settings menu with the generation of the path to `route ('platform.xsetting.list')` (for example, in the "System" menu), for this we will create a file `src / Providers / MenuComposer.php`
```
namespace Orchids\XSetting\Providers;

use Orchid\Platform\Dashboard;
class MenuComposer
{
    public function __construct(Dashboard $dashboard)
    {
        $this->dashboard = $dashboard;
    }

    public function compose()
    {
        $this->dashboard->menu
            ->add('CMS',
                ItemMenu::Label(__('Setting configuration'))
                    ->Slug('XSetting')              // Unique string containing only safe characters
                    ->Icon('icon-settings')         // CSS code for the graphic icon
                    ->title(__('Setting description'))  // Menu title
                    ->Route('platform.xsetting.list') // Path, route () or link
                    ->Permission('platform.systems.xsetting') // What rights the user should have
                    ->Sort(7)           // Sort menu items 1/2/3/4
            );
    }
}
```


#### 5. Now you need to add the `options` column to the database containing additional parameters of this key, for this we will create a database migration file
`database / migrations / 2018_08_07_000000_create_options_for_settings_table.php` containing
```
public function up()
{
    Schema::table('settings', function (Blueprint $table) {
        $table->jsonb('options');
    });
}
public function down()
{
    Schema::dropIfExists('settings', function (Blueprint $table) {
        $table->dropColumn('options');
    });
}
```


#### 6. You also need to create a model that describes the connections with the database, create a file src / Models / XSetting.php
```
namespace Orchids\XSetting\Models;

use Illuminate\Support\Facades\Cache;
use Orchid\Setting\Setting;
use Orchid\Platform\Traits\MultiLanguage;
use Orchid\Screen\AsMultiSource;

class XSetting extends Setting
{
    use Filterable, AsMultiSource;
    
    protected $fillable = ['key','value','options' ];	
    
    protected $casts = [
        'key' =>'string',
        'value' => 'array',
        'options' => 'array',
    ];	

}
```

#### 7. Now it remains to add screens [Screens] (https://orchid.software/ru/docs/screens) and layouts [Layouts] (https://orchid.software/en/docs/layouts)
Let's add a screen for the list of all settings, for this we will create a file src / Http / Screens / XSettingList.php
```
namespace Orchids\XSetting\Http\Screens;

use Orchid\Screen\Screen;
use Orchid\Screen\Layouts;
use Orchid\Screen\Actions\Button;

use Orchids\XSetting\Models\XSetting;
use Orchids\XSetting\Http\Layouts\XSettingListLayout;

class XSettingList extends Screen
{
    public $name = 'Setting List';
    public $description = 'List all settings';

    public function query() : array
    {
        return [
            'settings' => XSetting::filters()->defaultSort('key', 'desc')->paginate(30)  // The variable `settings` will be processed in the layout.
        ];
    }

    public function layout() : array
    {
        return [
            XSettingListLayout::class,   // Layout class.
        ];
    }
    
    public function commandBar() : array
    {
        return [
            Button::make('Create a new setting')->method('create'),  // Add an item for adding a setting to the top menu
            // will run the create () function
        ];
    }

     public function create()
    {
        return redirect()->route('platform.xsetting.create'); 
    }
}
```
More about creating screens [link] (https://orchid.software/ru/docs/screens)

#### 8. Add a layout for displaying a list of all settings, for this we will create a file src / Http / Layouts / XSettingListLayout.php 

```
namespace Orchids\XSetting\Http\Layouts;

use Orchid\Screen\Layouts\Table;
use Orchid\Screen\Fields\TD;

class XSettingListLayout extends Table
{
    public $data = 'settings';
    public function columns() : array
    {
        return  [
            TD::set('key','Key')         // Set variable name and column header
                ->link('platform.xsetting.edit','key','key'), / add a link, Parameters (link for routing, routing options, display text)
            TD::set('options.title', 'Name')
                ->setRender(function ($shortvar) {          // setRender - the function generates the rendered data
                    return $shortvar->options['title'];
                }),
            TD::set('value','Value')
                ->setRender(function ($xsetting) {
                     if (is_array($xsetting->value)) {
                        return \Str::limit(htmlspecialchars(json_encode($xsetting->value)), 50);
                     }
                     return \Str::limit(htmlspecialchars($xsetting->value), 50);
				}),
        ];
    }
}
```
More about creating layouts [link] (https://orchid.software/ru/docs/layouts)

#### 9. Add a screen for creating and editing settings, for this we will create a file src / Http / Screens / XSettingEdit.php
```
<?php
namespace Orchids\XSetting\Http\Screens;

use Orchid\Support\Facades\Alert;
use Orchid\Support\Facades\Setting;
use Orchid\Screen\Layouts;
use Orchid\Screen\Actions\Link;
use Orchid\Screen\Screen;

use Orchids\XSetting\Models\XSetting;
use Orchids\XSetting\Http\Layouts\XSettingEditLayout;

class XSettingEdit extends Screen
{
	
    public $name = 'Setting edit';
    public $description = 'Edit setting';

    public function query($xsetting = null) : array
    {
        $xsetting = is_null($xsetting) ? new XSetting() : XSetting::where("key",$xsetting)->first();;
        return [
            'xsetting'   => $xsetting,   // The variable `xsetting` will be processed in the layout.
        ];
    }

    public function layout() : array
    {
        return [
            Layouts::columns([
                'EditSetting' => [
                    XSettingEditLayout::class   // Layout that will process the data
                ],
            ]),
        ];
    }

    public function commandBar() : array
    {
        return [				
            Link::make('Save')->method('save'),   / Add the save option to the top menu, the save function will be processed by the `save` function
            Link::make('Remove')->method('remove'), // Add a setting removal item to the top menu will be processed by the `remove` function
        ];
    }

    public function save($request, XSetting $xsetting)   // Save the setting
    {

        $req = $this->request->get('xsetting');   // Filled data 'xsetting' "return" from layout

        $xsetting->updateOrCreate(['key' => $req['key']], $req );

        Alert::info('Setting was saved');
        return back();
    }
	 
    public function remove($request, XSetting $xsetting)  // Function for removing the setting
    {
        $xsetting->where('id',$request)->delete();
        Alert::info('Setting was removed');
        return back();
    }
}
```
#### 10. It remains to add a layout for editing settings, create `src / Http / Layouts / XSettingEditLayout.php`
```
namespace Orchids\XSetting\Http\Layouts;

use Orchid\Screen\Layouts\Rows;
use Orchid\Screen\Fields\Field;
use Orchid\Screen\Fields\Builder;

class XSettingEditLayout extends Rows
{
	public function fields(): array
    {
        return [
            Field::tag('input')				// Input field
                ->name('xsetting.key')		
                ->required()				// Required
                ->max(255)
                ->title('Key slug'),		// Title
            Field::tag('input')
                ->name('xsetting.options.title')
                ->required()
                ->max(255)
                ->title('Title'),
            Field::tag('textarea')
                ->name('xsetting.options.desc')
                ->row(5)
                ->title('Description'),
            Field::tag('textarea')
                ->name('xsetting.value')
                ->title('Value'),
        ];
	}
}
```

More about fields [link] (https://orchid.software/ru/docs/field)

#### 11. Installation
- Run in the console `composer require" orchids / xsetting "`
- Apply migrations `php artisan migrate`
- In the admin panel, assign access to the user.

[Plugin link on github] (https://github.com/orchidcommunity/XSetting)

#### Update from 12-08-18
The plugin has been added the ability to select the type of stored data:
- Input - string
- Textarea - Text
- Picture - Image (For example, company logo)
- CodeEditor (JSON) - any JSON array.
- CodeEditor (JavaScript) - JavaScript, HTML code (For example, google analytics code).
- Tags - a list of words, tags (For example, keywords in `meta name =" keywords "`).

![](https://github.com/orchidcommunity/XSetting/blob/master/docs/imgs/create_plugin2.gif)
