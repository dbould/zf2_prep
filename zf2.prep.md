# ZF2 Certified Architect Notes & Prep

## Modules

The module manager is the component that allows for modules to be loaded
and reused. Modules provide configuration which are loaded in a logic
order so you can override specific behaviors.

### Module Manager

**Events:**

loadModules - triggered at start of ModuleManager::loadModules()
loadModule.resolve - triggered in ModuleManager::loadModuleByName()
loadModule - triggered each time a module is loaded
loadModules.post - triggered at end of ModuleManager::loadModules()

**Interface:**

ModuleManagerInterface

* loadModules
* loadModule
* getLoadedModules
* getModules
* setModules

**Sample config:**

These are various options for the listeners attached to the ModuleManager.

Typically found in config/application.config.php

```php
[
    modules => [
        'Application', // The modules to be loaded by manager
    ],
    'module_listener_options' => [
        'module_paths' => [
            './module', // Applications dir to load modules from
            './vendor' // Vendor dir to load modules from
        ],
        'config_glob_paths' => [
            'config/autoload/{,*.}{global,local}.php', // config autoload rule
        ]
    ],
]
```

If using the skeleton application the modules will be loaded during:

```php
    Zend\Mvc\Application::init(require 'config/application.config.php')->run();
```

Which internally runs:

```php
    $serviceManager->get('ModuleManager')->loadModules();
```

### Modules

Each ZF2 module will have a Module.php file in the root of the namespace.
The root namespace is taken as the name of the module (i.e. in config array).

**Sample Module.php**

```php
<?php
namespace Application;

class Module
{

}
```

Module.php files can be implement various interfaces to extend functionality.

See Zend\ModuleManager\Feature namespace in zend-modulemanager for a full list.

Often used interfaces

* ConfigProviderInterface
* BootstrapListenerInterface
* ServiceProviderInterface

Configuration files from all the modules are merged together into one array.
This can then be cached provided it does not contain closures.

Configurations are merged in order of:

* Array Module::getConfig() returns and config/module.config.php
* Global autoloads - config/autoload/global.*.php
* Local autoloads - config/autoload/local.*.php
* Specific feature Configs i.e. Module::getServiceConfig()

The first 3 arrays will be merged and cached, whereas specific feature configs
will not so that closures can be used in these.

## Service Locator

The service manager implements the service locator pattern. It is used for
dependency injection and managing dependencies. It's primary concern is to
instantiate classes.

Service Locators must implement Zend\ServiceManager\ServiceLocatorInterface

Which contains to methods:

* ServiceLocatorInterface::get($name);
* ServiceLocatorInterface::has($name);

### Service Manager

When getting a service from a service manager the $name argument is normalized,
so any of the following would retrieve the same service;

```php
<?php
    $services->get('My-Service');
    $services->get('myservice');
    $services->get('MyService');
```

Service will be lazy loaded and classes will only be instantiated from
invokables or factories when get method is called on ServiceManager and
provided with matching name as first argument.

**Example Configs**

```php
[
    'service_manager' => [
        'invokables' => [
            'MyNamespace\\MyService' => 'MyNamespace\\Service\\MyService',
        ],
        'factories' => [
            'MyNamespace\\MyOtherService' => 'MyNamespace\\Factory\\MyServiceFactory',
        ]
    ]
]
```

**Abstract Factories**

Abstract Factories should implement the AbstractFactoryInterface. Which
provides the following method signatures:

* canCreateServiceWithName(ServiceLocatorInterface $serviceLocator, $name, $requestedName)
* createServiceWithName(ServiceLocatorInterface $serviceLocator, $name, $requestedName)

**Initializers**

Initializers can be used to listen for a certain interface and then run
specific logic using a closure

For Example:

```php
[
    'initializers' => [
        'CalculatorInit' => function($service, $serviceManager) {
            if ($service instanceof CalculatorAwareService) {
                $service->setThing($serviceManager->get('thing'));
            }
        }
    ]
]
```

### Plugin Managers

Core compontents of the framework will have there own implementation of the
ServiceManager these will all extend the AbstractPluginManager class which in
turn extends ServiceManager class.

The AbstractPluginManager also implements the ServiceLocatorAwareInterface so
by calling Plugin::getServiceLocator() you can get an instance of the
ServiceManager.

Key Plugin Managers:

* ControllerPluginManager
* ControllerManager
* FormElementManager
* HydratorManager
* ViewHelperManager
* ValidatorManager

The AbstractPluginManager has the following abstract method which must be
implemented by sub classes:

```php
abstract public function validatePlugin($plugin);
```

#### ControllerManager

The controller manager is used internally by ZF2 during the routing process.

The matched route name is used to get an instance of the correct controller
out of the Manager.

**Example Config**

```php
[
    'controllers' => [
        'invokables' => [
            'MyController' => 'My\\Namespace'
        ]
    ],
    'router' => [
        'routes' => [
            'home' => [
                'type' => 'Zend\\Mvc\\Router\\Http\\Literal',
                'options' => [
                    'route'    => '/my-route',
                    'defaults' => [
                        'controller' => 'MyController',
                        'action'     => 'index',
                    ],
                ],
            ],
        ]
    ]
]
```
