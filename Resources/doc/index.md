Using KnpMenuBundle
===================

Welcome to KnpMenuBundle - creating menus is fun again!

* [Installation](#installation)
* [Your first menu](#first-menu)

<a name="installation"></a>

## Installation

### Step 1) Get the bundle and the library

First, grab the KnpMenu library and KnpMenuBundle. There are two different ways
to do this:

#### Method a) Using the `deps` file

You can also achieve the same by using the `deps` file. Simply add the new
vendors to your `deps` file and then run `php bin/vendors install`:

```
[knp-menu]
    git=https://github.com/knplabs/KnpMenu.git

[KnpMenuBundle]
    git=https://github.com/knplabs/KnpMenuBundle.git
    target=bundles/Knp/Bundle/MenuBundle
```

#### Method b) Using submodules

Run the following commands to bring in the needed libraries as submodules.

```bash
git submodule add https://github.com/knplabs/KnpMenuBundle.git vendor/bundles/Knp/Bundle/MenuBundle
git submodule add https://github.com/knplabs/KnpMenu.git vendor/knp-menu
```

### Step 2) Register the namespaces

Add the following two namespace entries to the `registerNamespaces` call
in your autoloader:

``` php
<?php
// app/autoload.php
$loader->registerNamespaces(array(
    // ...
    'Knp\Bundle' => __DIR__.'/../vendor/bundles',
    'Knp\Menu'   => __DIR__.'/../vendor/knp-menu/src',
    // ...
));
```

### Step 3) Register the bundle

To start using the bundle, register it in your Kernel. This file is usually
located at `app/AppKernel.php`:

``` php
<?php
// app/AppKernel.php

public function registerBundles()
{
    $bundles = array(
        // ...
        new Knp\Bundle\MenuBundle\KnpMenuBundle(),
    );
    // ...
)
```

### Step 4) Configure the bundle

```yaml
# app/config/config.yml
knp_menu:
    twig: false  # disables the Twig extension and the TwigRenderer
    templating: true # enables the helper for PHP templates
    default_renderer: list # Change the default renderer as we disabled the Twig one
```

>**NOTE**
>The configuration is optional. If you omit it, the default behavior is to
>enable the Twig support, to disable the PHP helper (as Twig is the recommended
>templating engine in Symfony2) and to use the Twig renderer as default renderer.

<a name="first-menu"></a>

## Create your first menu!

Suppose that from your layout, you'd like to render a menu in your sidebar.
One way to do this is to render a controller from our `AcmeMainBundle`, which
will handle the work:

```jinja
<div id="sidebar">
    {% render 'AcmeMainBundle:Default:sidebar' %}
</div>
```

In the corresponding controller, we'll create and render the menu:

``` php
<?php
// src/Acme/MainBundle/Controller/DefaultController.php

>**NOTE**
>Registering your menu in the menu provider is optional. You could also create
>the menu in your controller and pass it explicitly to your template.

In this example, we use the `knp_menu.factory` service to create a new `MenuItem`
object, which we can then customize. Later we use the `knp_menu.renderer.twig`
service to actually render the menu. We pass the rendered menu into a `Response`
object and are done!

Of course, there's any easier way! By configuring your menus as services,
you can easily render them right inside a template, without needed to render
a controller.

## Registering a menu in the provider

A good way to build a menu is in a centralized builder class. This will ultimately
make it very easy to render any menu.

Start by creating a builder for your menu. You can stick as many menus into
a builder as you want, so you may only have one of these builder classes
in your application:

```php
<?php
// src/Acme/MainBundle/Menu/MenuBuilder.php

namespace Acme\MainBundle\Menu;

use Knp\Menu\FactoryInterface;
use Symfony\Component\HttpFoundation\Request;

class MenuBuilder
{
    private $factory;

    /**
     * @param FactoryInterface $factory
     */
    public function __construct(FactoryInterface $factory)
    {
        $this->factory = $factory;
    }

    public function createMainMenu(Request $request)
    {
        $menu = $this->factory->createItem('root');
        $menu->setCurrentUri($request->getRequestUri());

        $menu->addChild('Home', array('route' => 'homepage'));
        // ... add more children

        return $menu;
    }
}
```

Next, register two services: one for your menu builder, and one for the menu
object created by the `createMainMenu` method:

```yaml
# src/Acme/MainBundle/Resources/config/services.yml
services:
    acme_main.menu_builder:
        class: Acme\MainBundle\Menu\MenuBuilder
        arguments: ["@knp_menu.factory", "@router"]

    acme_main.menu.main:
        class: Knp\Menu\MenuItem # the service definition requires setting the class
        factory_service: acme_hello.menu_builder
        factory_method: createMainMenu
        arguments: ["@request"]
        scope: request # needed as we have the request as a dependency here
        tags:
            - { name: knp_menu.menu, alias: main } # The alias is what is used to retrieve the menu
```

>**NOTE**
>The menu service must be public as it will be retrieved at runtime to keep
>it lazy-loaded.

You can now retrieve the menu by its name in your template:

You can now render the menu directly in a template via the name given in the
`alias` key above:

```jinja
{{ knp_menu_render('main') }}
```

Suppose now we need to create a second menu for the sidebar. The process
is simple! Start by adding a new method to your builder:

```php
<?php
// src/Acme/MainBundle/Menu/MenuBuilder.php

// ...

class MenuBuilder
{
    // ...

    public function createSidebarMenu(Request $request)
    {
        $menu = $this->factory->createItem('sidebar');
        $menu->setCurrentUri($request->getRequestUri());

        $menu->addChild('Home', array('route' => 'homepage'));
        // ... add more children

        return $menu;
    }
}
```

Now, create a service for *just* your new menu, giving it a new name, like
`sidebar`:

```yaml
# src/Acme/MainBundle/Resources/config/services.yml
services:

    acme_main.menu.sidebar:
        class: Knp\Menu\MenuItem
        factory_service: acme_hello.menu_builder
        factory_method: createSidebarMenu
        arguments: ["@request"]
        scope: request
        tags:
            - { name: knp_menu.menu, alias: sidebar } # Named "sidebar" this time
```

It can now be rendered, just like the other menu:

## Registering your own renderer

Registering your own renderer in the renderer provider is simply a matter
of creating a service tagged with `knp_menu.renderer`:

```jinja
{{ 'main'|knp_menu_render('twig') }}
```

```yaml
# src/Acme/MainBundle/Resources/config/services.yml
services:
    acme_hello.menu_renderer:
        class: Acme\MainBundle\Menu\CustomRenderer # The class implements Knp\Menu\Renderer\RendererInterface
        arguments: [%kernel.charset%] # set your own dependencies here
        tags:
            - { name: knp_menu.renderer, alias: custom } # The alias is what is used to retrieve the menu
```

>**Note**
>The renderer service must be public as it will be retrieved at runtime to
>keep it lazy-loaded.

You can now use your renderer to render your menu:

```jinja
{{ knp_menu_render('main', {'my_custom_option': 'some_value'}, 'custom') }}
```

>**NOTE**
>As the renderer is responsible to render some HTML code, the `knp_menu_render`
>filter is marked as safe. Take care to handle escaping data in your renderer
>to avoid XSS if you use some user input in the menu.

## Using PHP templates

If you prefer using PHP templates, you can use the templating helper to render
and retrieve your menu from a template, just like available in Twig.

```php
// Retrieves an item by its path in the main menu
$item = $view['knp_menu']->get('main', array('child'));

// Render an item
echo $view['knp_menu']->render($item, array(), 'list');
```
