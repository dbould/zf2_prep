# ZF2 Certified Architect Notes & Prep

## Modules

The module manager is the component that allows for modules to be loaded
and reused. Modules provide configuration which are loaded in a logic
order so you can override specific behaviors.

### Module Manager

**Events:**

* loadModules - triggered at start of ModuleManager::loadModules()
* loadModule.resolve - triggered in ModuleManager::loadModuleByName()
* loadModule - triggered each time a module is loaded
* loadModules.post - triggered at end of ModuleManager::loadModules()

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
    'modules' => [
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

## Event Manager

The event manager is an implementation of the observer pattern.

Default Events:

* bootstrap
* route
* [dispatch.error]
* dispatch
* [dispatch.error]
* render
* [render.error]
* finish

**The EventManagerInterface**

```php
    public function attach($event, $callback = null, $priority = 1);

    public function trigger($event, $target = null, $argv = [], $callback = null);
```

**Provider:**
A provider in ZF2 is any class which triggers an event via the event manager.

**Observer**

In ZF2 an observer is a listener attached via the event manager as a callback.

### Aspect Orientation

Aspect orientation helps deal with cross cutting concerns. In compiled langauges
it is easy to insert code into methods during compilation solving this problem.

Since PHP is not compiled other solutions have been implemented. One of these
being the ZF2 event manager using pre and post events in a method.

**For Example:**

```php
<?php
namespace MyNamespace;

class Tester
{
    public function testIt()
    {
        $this->eventManager-addIdentifiers(array(
            get_called_class()
        ));
        $this->eventManager->trigger('testIt.pre', $this, ['test']);

        // Some logic

        $this->eventManager->trigger('testIt.post', $this, ['test']);
    }
}
```

```php
class Module
{
    public function onBootstrap(MvcEvent $event)
    {
        $eventManager = $event->getApplication()->getEventManager();
        $sharedEventManager = $eventManager->getSharedManager();

        $sharedEventManager->attach('MyNamespace', 'testIt.pre', function($e) {
            echo 'Test it';
        }, 100);
    }
}
```

## Zend DB

An abstraction layer to allow for databases to be queried

### Adapter

To use Zend DB an adapter must be instantiated taking config of database
credentials and details as first argument.

**DriverInterface**

```php
<?php
namespace Zend\Db\Adapter\Driver;

 interface DriverInterface
 {
     const PARAMETERIZATION_POSITIONAL = 'positional';
     const PARAMETERIZATION_NAMED = 'named';
     const NAME_FORMAT_CAMELCASE = 'camelCase';
     const NAME_FORMAT_NATURAL = 'natural';
     public function getDatabasePlatformName($nameFormat = self::NAME_FORMAT_CAMELCASE);
     public function checkEnvironment();
     public function getConnection();
     public function createStatement($sqlOrResource = null);
     public function createResult($resource);
     public function getPrepareType();
     public function formatParameterName($name, $type = null);
     public function getLastGeneratedValue();
 }
 ```

**Example Adapter**

```php
$dbAdapter = new Adapter([
    'driver' => 'Pdo',
    'dsn' => 'mysql:dbname=example_database;host=localhost',
    'username' => 'test',
    'password' => 'testpass',
]);
```

**Using the adapter**

```php
// createStatement
$sql = 'SELECT * FROM users;';
$statement = $adapter->createStatement($sql);
$result = $statement->execute();

// query
$sql = 'SELECT * FROM users WHERE `id` = ?;';
$result = $adapter->query($sql, [2]);
```

### Db\\Sql

Compontent to allow SQL queries to be built up programmatically.

**Example**

```php
<?php
use Zend\Db\Sql\Sql;

$sql = new Sql($adapter);
$select = $sql->select(); // @return Zend\Db\Sql\Select
$insert = $sql->insert(); // @return Zend\Db\Sql\Insert
$update = $sql->update(); // @return Zend\Db\Sql\Update
$delete = $sql->delete(); // @return Zend\Db\Sql\Delete
```

**Example Select**

```php
use Zend\Db\Sql\Sql;
$sql = new Sql($adapter);
$select = $sql->select();
$select->from('foo');
$select->where(array('id' => 2));

$statement = $sql->prepareStatementForSqlObject($select);
$results = $statement->execute();
```

### Data Definition Language

A component to handle SQL operations such as creation and alteration of tables
and index etc.

```php
use Zend\Db\Sql\Ddl;

$table = new Ddl\CreateTable();

// or with table
$table = new Ddl\CreateTable('bar');

// optionally, as a temporary table
$table = new Ddl\CreateTable('bar', true);
```

### Table Gateway

The table gateway can be used to execute queries against a specific table.

**TableGatewayInterface**

```php
<?php
namespace Zend\Db\TableGateway;

interface TableGatewayInterface
{
    public function getTable();
    public function select($where = null);
    public function insert($set);
    public function update($set, $where = null);
    public function delete($where);
}
```

**Example Query**

```php
use Zend\Db\TableGateway\TableGateway;
$projectTable = new TableGateway('project', $adapter);
$rowset = $projectTable->select(array('type' => 'PHP'));
```

## Logger

Code below will register Logger before the current PHP error handler, and will
pass the error along as well.

```php
Zend\Log\Logger::registerErrorHandler($logger);
```

**Writers**

* \\Zend\\Log\\Writer\\Db - logs to database
* \\Zend\\Log\\Writer\\Stream - filesystem or php://output

## Web Services

### Http Client

**DispatchableInterface**

```php
public function dispatch(RequestInterface $request, ResponseInterface $response = null);
```

**Http Client example**

```php
$this->httpClient->reset();

$headers = new \Zend\Http\Headers();
$this->httpClient->setUri($this->formConfigApi['form_configuration_url']);
$this->httpClient->setMethod(Request::METHOD_GET);

$headers->addHeaderLine("Accept", "application/json");
$headers->addHeaderLine("Content-Type", "application/json");
$headers->addHeaderLine("Authorization", "bearer " . $this->session->access_token);
$this->httpClient->setHeaders($headers);

$response = $this->httpClient->send();
```

### REST

* GET {/route} - AbstractRestfulController::getList()
* GET {/route/[:id]} - AbstractRestfulController::get($id)
* PUT - AbstractRestfulController::update($id, $data)
* POST - AbstractRestfulController::create($data)
* DELETE - AbstractRestfulController::delete($id)

**Json Strategy**

```php
'view_manager' => array(
    'strategies' => array(
        'ViewJsonStrategy',
    ),
),
```

Once the above code is in place returning a JsonModel from view will be converted
to JSON.

**AcceptableViewModelSelector**

AcceptableViewModelSelector can be used to return different View Models based
on Content-Type headers.

### SOAP

SOAP client consumes an endpoint

**SOAP Client Example**

```php
<?php

use Zend\Soap\Client;

include __DIR__ . '/vendor/autoload.php';

// Modify the endpoint as necessary
$client = new Client('http://localhost:8080/soap.php');

// Modify '$original' to test encoding
$original = 'Hello world!';
echo "Unencoded '$original': ";
echo $client->encode($original), "\n"; // Method comes from setClass argument interface

// Modify '$encoded' to test decoding
$encoded = str_rot13('Hello world!');
echo "Encoded '$encoded': ";
echo $client->decode($encoded), "\n";  // Method comes from setClass argument interface
```

Sop

**SOAP Server Example**

Using the AutoDiscover set handler using AutoDiscover::setClass method.

```php
<?php

use Zend\Http\PhpEnvironment\Request;
use Zend\Soap\AutoDiscover;
use Zend\Soap\Server;

include __DIR__ . '/vendor/autoload.php';

require_once __DIR__ . '/Rot13.php';

$uri     = 'http://localhost:8080/soap.php';
$request = new Request();

switch ($request->getMethod()) {
    case Request::METHOD_GET:
        $server = new AutoDiscover();
        $server->setClass('Application\Rot13') // Set the class for client API
            ->setUri($uri)
            ->setServiceName('Rot13');
        $wsdl = $server->generate();
        echo $wsdl->toXml();
        break;
    case Request::METHOD_POST:
        $server = new Server($uri);
        $server->setClass('Zf2AdvancedCourse\Rot13');
        $server->handle();
        break;
    default:
        header('HTTP/1.1 405 Method Not Allowed');
        exit(1);
}
```
## Forms

Form extends Fieldset extends Element

**Example element config**

```php
[
    'name' => 'Gender',
    'type' => 'Select',
    'options' => [
        'label' => 'GENDER',
        'empty_option' => 'Please select',
        'value_options' => [
            1 => 'Female'
            2 => 'Male',
        ]
    ]
]
```

**Example element**

```php
use Zend\Form\Form;
    $form = new Form('my_form');
    $form->add(array(
            'type' => 'Zend\Form\Element\Select',
            'name' => 'language',
            'options' => array(
                    'label' => 'Which is your mother tongue?',
                    'value_options' => array(
                            '0' => 'French',
                            '1' => 'English',
                            '2' => 'Japanese',
                            '3' => 'Chinese',
                    ),
            )
    ));
```

**Get and Set Data**

```php
    $form = new Form();
    $form->setData($data);

    if ($form->isValid === true) {
        $filteredData = $form->getData();
    }
```

**Get and Set Data with entity**

```php
    $form = new Form();
    $form->setObject(new UserEntity()); // this would be done in Form Class
    $form->bind(new UserEntity()); // this would be done in controller

    $form->setData($userData);

    if ($form->isValid === true) {
        $user = $form->getObject();
    }
```

**Rendering forms in view phtml**

```php
<?php $this->form->prepare(); // Always call prepare in view ?>

<?php $this->form($this->form); // Render whole form ?>
<?php $this->formRow($this->form->get('Gender')); // Render an element ?>

```

## Authentication

### RBAC

Role based access control.

* Roles have defined permissions
* Roles have inheritance


**Roles**

Roles can be created by implementing Zend\\Permissions\\Rbac\\AbstractRole

**AssertionInterface**

The AssertionInterface can be used to create callbacks to supplement roles.

For example if you required a User to also be part of a subscription model an
assertion could be used.

## Pagination

The paginator class takes an adapter as first argument to decouple the Pagination
code from any specific data source.

**Default Adapters**

* ArrayAdapter
* DbSelect
* DbTableGateway
* Iterator
* Callback

**AdapterInterface**

```php
/**
 * Returns a collection of items for a page.
 *
 * @param  int $offset Page offset
 * @param  int $itemCountPerPage Number of items per page
 * @return array
 */
public function getItems($offset, $itemCountPerPage);
```

**Example**

```php
$data = ['this', 'that'];

$paginator = new Paginator(new ArrayAdapter($data));

```

**Paginator in view**
```php
<ul>
<?php foreach ($this->pagiantor as $item) : ?>
    <li><?php echo $item;?></li>
<?php endforeach;?>
</ul>
```

**Paginator Controls**
```php
<?php
// add at the end of the file after the table
echo $this->paginationControl(
    // the paginator object
    $this->paginator,
    // the scrolling style
    'sliding',
    // the partial to use to render the control
    'partial/paginator.phtml',
    // the route to link to when a user clicks a control link
    array(
        'route' => 'album'
    )
);
?>
```

## Mail

**Send Message**

```php
$message = new Message();

$message->setBody('This is a test');
$message->setSubject('Test Subject');
$message->setFrom('example@example.com', 'Example Name');
$message->addTo('receipient@example.com', 'Recipient Name');

$transport = new SendMail();
$transport->send($message);
```

**Transports**

* SendMail
* Smtp
* File

## Hydrators

Hydrators allow data stored in an array to be transposed into an object.

ZF2 Ships with various default Hydrators including:

**Interfaces**

```php
interface HydrationInterface
{
    /**
     * Hydrate $object with the provided $data.
     *
     * @param  array $data
     * @param  object $object
     * @return object
     */
    public function hydrate(array $data, $object);
}
```

```php
interface ExtractionInterface
{
    /**
     * Extract values from an object
     *
     * @param  object $object
     * @return array
     */
    public function extract($object);
}
```

**Filtering**

Filter interface

```php
interface FilterInterface
{
    /**
     * Returns the result of filtering $value
     *
     * @param  mixed $value
     * @throws Exception\RuntimeException If filtering $value is impossible
     * @return mixed
     */
    public function filter($value);
}
```
