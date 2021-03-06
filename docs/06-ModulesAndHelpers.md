---
layout: doc
title: 06-ModulesAndHelpers - Codeception - Documentation
---

# Modules and Helpers

Codeception uses modularity to create a comfortable testing environment for every test suite you write.

All actions and assertions that can be performed by the Tester object in a class are defined in modules. You can extend the testing suite with your own actions and assertions, by writing them into a custom module.

Let's look at the following test:

{% highlight php %}

<?php
$I = new FunctionalTester($scenario);
$I->amOnPage('/');
$I->see('Hello');
$I->seeInDatabase('users', array('id' => 1));
$I->seeFileFound('running.lock');


{% endhighlight %}

It can operate with different entities: the web page can be loaded with the PhpBrowser module, the database assertion uses the Db module, and file state can be checked with the Filesystem module. 

Modules are attached to Actor classes in the suite config.
For example, in `tests/functional.suite.yml` we should see:

{% highlight yaml %}

class_name: FunctionalTester
modules:
    enabled: 
        - PhpBrowser:
            url: http://localhost
        - Db:
            dsn: "mysql:host=localhost;dbname=testdb"
        - Filesystem

{% endhighlight %}

The FunctionalTester class has its methods defined in modules. Actually it doesn't contain any of them, but rather acts as a proxy. It knows which module executes this action and passes parameters into it. To make your IDE see all of the FunctionalTester methods, you should run use the `codecept build` command. It generates method signatures from enabled modules and saves them into a trait which is included into an actor. In current example, `tests/support/_generated/FunctionalTesterActions.php` file will be generated.
By default Codeception auto rebuilds Actions trait on each change of suite configuration.

## Standard Modules

Codeception has many bundled modules which will help you run tests for different purposes and different environments. The idea of modules is to share common actions so developers and QA engineers could concentrate on testing and not on reinventing the wheel. Each module provides methods for testing its part, by combining modules you can get powerful setup to test application at all levels.

There is `WebDriver` module for acceptance testing, modules for all popular PHP frameworks, `PHPBrowser` to emulate browser execution, `REST` for testing APIs, and more. Modules are considered to be the most valuable part of Codeception. They are constanly improving to provide the best testing experience, and be flexible to satisfy everyone's needs.

### Module Conflicts

Modules may conflict with each other. If a module implements `Codeception\Lib\Interfaces\ConflictsWithModule` it might declare a conflict rule to be used with other modules. For instance, WebDriver condlicts with all modules implementing `Codeception\Lib\Interfaces\Web` interface.

{% highlight php %}

public function _conflicts()
{
    return 'Codeception\Lib\Interfaces\Web';
}

{% endhighlight %}

This way if you try to use two modules sharing the same conflicted interface you will get an exception.

**Framework modules, PhpBrowser, and WebDriver** can't be used together to avoid confusion. For instance, `amOnPage` method exists in all those modules, and you should not guess which module will actually execute it. If you do acceptance testing set up either WebDriver or PHPBrowser but do not setup both. If you do functional testing, enable one of the framework modules. 

In case you need to use a module which depends on conflicted one, specify it as dependent module in config. Probably you may want to use `WebDriver` with `REST` module which interacts with a server through `PhpBrowser`. In this case your config should look like:

{% highlight yaml %}

modules:
    enabled:
        - WebDriver:
            browser: firefox
            url: http://localhost
        - REST:
            url: http://localhost/api/v1
            depends: PhpBrowser


{% endhighlight %}

This config will allow you to send GET/POST requests to server's APIs while working with a site through a browser.

In case you only need some parts of conflicted module to be loaded, please refer to the next section.

### Module Parts

Modules with *Parts* section in their reference can be partially loaded. This way `$I` object will have actions belonging only to a specific part of that module. Partially loaded modules can be also used to avoid module conflicts.

For instance, Laravel5 module has ORM part which contains database actions. You can enable PhpBrowser module for testing and Laravel + ORM for connecting to database and checking the data.

{% highlight yaml %}

modules:
    enabled:
        - PhpBrowser:
            url: http://localhost
        - Laravel5:
            part: ORM            

{% endhighlight %}

Modules won't conflict as actions with the same names won't be loaded.

The same way REST module has `Xml` and `Json` parts. In case you test REST service with JSON responses only, you can enable only JSON part of this module:

{% highlight yaml %}

class_name: ApiTester
modules:
    enabled:
        - REST:
            url: http://serviceapp/api/v1/
            depends: PhpBrowser
            part: Json    

{% endhighlight %}

## Helpers

Codeception doesn't restrict you to only the modules from the main repository. No doubt your project might need your own actions added to the test suite. By running the `bootstrap` command, Codeception generates three dummy modules for you, one for each of the newly created suites. These custom modules are called 'Helpers', and they can be found in the `tests/_support` directory.



{% highlight php %}

<?php
namespace Helper;
// here you can define custom functions for FunctionalTester

class Functional extends \Codeception\Module
{
}


{% endhighlight %}

As for actions, everything is quite simple. Every action you define is a public function. Write any public method, run the `build` command, and you will see the new function added into the FunctionalTester class.


<div class="alert alert-info">
Public methods prefixed by `_` are treated as hidden and won't be added to your Actor class.
</div>

Assertions can be a bit tricky. First of all, it's recommended to prefix all your assert actions with `see` or `dontSee`.

Name your assertions like this:

{% highlight php %}

<?php
$I->seePageReloaded();
$I->seeClassIsLoaded($classname);
$I->dontSeeUserExist($user);


{% endhighlight %}
And then use them in your tests:

{% highlight php %}

<?php
$I->seePageReloaded();
$I->seeClassIsLoaded('FunctionalTester');
$I->dontSeeUserExist($user);


{% endhighlight %}

You can define asserts by using assertXXX methods in modules.

{% highlight php %}

<?php

function seeClassExist($class)
{
    $this->assertTrue(class_exists($class));
}


{% endhighlight %}

In your helpers you can use these assertions:

{% highlight php %}

<?php

function seeCanCheckEverything($thing)
{
    $this->assertTrue(isset($thing), "this thing is set");
    $this->assertFalse(empty($any), "this thing is not empty");
    $this->assertNotNull($thing, "this thing is not null");
    $this->assertContains("world", $thing, "this thing contains 'world'");
    $this->assertNotContains("bye", $thing, "this thing doesn`t contain 'bye'");
    $this->assertEquals("hello world", $thing, "this thing is 'Hello world'!");
    // ...
}


{% endhighlight %}

### Accessing Other Modules

It's possible that you will need to access internal data or functions from other modules. For example, for your module you might need to access responses or internal actions of modules.

Modules can interact with each other through the `getModule` method. Please note that this method will throw an exception if the required module was not loaded.

Let's imagine that we are writing a module that reconnects to a database. It's supposed to use the dbh connection value from the Db module.

{% highlight php %}

<?php

function reconnectToDatabase() {
    $dbh = $this->getModule('Db')->dbh;
    $dbh->close();
    $dbh->open();
}


{% endhighlight %}

By using the `getModule` function, you get access to all of the public methods and properties of the requested module. The `dbh` property was defined as public specifically to be available to other modules.

Modules may also contain methods that are exposed for use in helper classes. Those methods start with `_` prefix and are not available in Actor classes, so can be accessed only from modules and extensions.

You should use them to write your own actions using module internals.
   
{% highlight php %}

<?php
function seeNumResults($num)
{
    // retrieving webdriver session
    /**@var $table \Facebook\WebDriver\WebDriverElement */
    $elements = $this->getModule('WebDriver')->_findElements('#result');
    $this->assertNotEmpty($elements);
    $table = reset($elements);
    $this->assertEquals('table', $table->getTagName());
    $results = $table->findElements('tr');
    // asserting that table contains exactly $num rows
    $this->assertEquals($num, count($results));
}


{% endhighlight %}

In this example we use API of <a href="https://github.com/facebook/php-webdriver">facebook/php-webdriver</a> library, a Selenium WebDriver client a module is build on. 
You can also access `webDriver` property of a module to get access to `Facebook\WebDriver\RemoteWebDriver` instance for direct Selenium interaction.

### Extending a Module

If accessing modules doesn't provide needed flexibility you can extend a module inside a Helper class:

{% highlight php %}

<?php
namespace Helper;

class MyExtendedSelenium extends \Codeception\Module\WebDriver  {
}


{% endhighlight %}

In this helper you can replace parent's methods with your own implementation.
You can also replace `_before` and `_after` hooks, which might be an option when you need to customize starting and stopping of a testing session.

If some of the methods of the parent class should not be used in a child module, you can disable them. Codeception has several options for this:

{% highlight php %}

<?php
namespace Helper;

class MyExtendedSelenium extends \Codeception\Module\WebDriver 
{
    // disable all inherited actions
    public static $includeInheritedActions = false;

    // include only "see" and "click" actions
    public static $onlyActions = ['see','click'];

    // exclude "seeElement" action
    public static $excludeActions = ['seeElement'];
}


{% endhighlight %}

Setting `$includeInheritedActions` to false adds the ability to create aliases for parent methods.
 It allows you to resolve conflicts between modules. Let's say we want to use the `Db` module with our `SecondDbHelper`
 that actually inherits from `Db`. How can we use `seeInDatabase` methods from both modules? Let's find out.

{% highlight php %}

<?php
namespace Helper;

class SecondDb extends \Codeception\Module\Db 
{
    public static $includeInheritedActions = false;

    public function seeInSecondDb($table, $data)
    {
        $this->seeInDatabase($table, $data);
    }
}


{% endhighlight %}

Setting `$includeInheritedActions` to false won't include the methods from parent classes into the generated Actor.
 
### Hooks

Each module can handle events from the running test. A module can be executed before the test starts, or after the test is finished. This can be useful for bootstrap/cleanup actions.
You can also define special behavior for when the test fails. This may help you in debugging the issue.
For example, the PhpBrowser module saves the current webpage to the `tests/_output` directory when a test fails.

All hooks are defined in `\Codeception\Module` and are listed here. You are free to redefine them in your module.

{% highlight php %}

<?php

    // HOOK: used after configuration is loaded
    public function _initialize() {
    }

    // HOOK: on every Actor class initialization
    public function _cleanup() {
    }

    // HOOK: before each suite
    public function _beforeSuite($settings = array()) {
    }

    // HOOK: after suite
    public function _afterSuite() {
    }    

    // HOOK: before each step
    public function _beforeStep(\Codeception\Step $step) {
    }

    // HOOK: after each step
    public function _afterStep(\Codeception\Step $step) {
    }

    // HOOK: before test
    public function _before(\Codeception\TestInterface $test) {
    }

    // HOOK: after test
    public function _after(\Codeception\TestInterface $test) {
    }

    // HOOK: on fail
    public function _failed(\Codeception\TestInterface $test, $fail) {
    }


{% endhighlight %}

Please note that methods with a `_` prefix are not added to the Actor class. This allows them to be defined as public but used only for internal purposes.

### Debug

As we mentioned, the `_failed` hook can help in debugging a failed test. You have the opportunity to save the current test's state and show it to the user, but you are not limited to this.

Each module can output internal values that may be useful during debug.
For example, the PhpBrowser module prints the response code and current URL every time it moves to a new page.
Thus, modules are not black boxes. They are trying to show you what is happening during the test. This makes debugging your tests less painful.

To display additional information, use the `debug` and `debugSection` methods of the module.
Here is an example of how it works for PhpBrowser:

{% highlight php %}

<?php
    $this->debugSection('Request', $params);
    $this->client->request($method, $uri, $params);
    $this->debug('Response Code: ' . $this->client->getStatusCode());
    

{% endhighlight %}

This test, running with the PhpBrowser module in debug mode, will print something like this:

{% highlight bash %}

I click "All pages"
* Request (GET) http://localhost/pages {}
* Response code: 200

{% endhighlight %}


## Configuration

Modules and Helpers can be configured from the suite config file, or globally from `codeception.yml`.

Mandatory parameters should be defined in the `$requiredFields` property of the class. Here is how it is done in the Db module:

{% highlight php %}

<?php
class Db extends \Codeception\Module 
{
    protected $requiredFields = ['dsn', 'user', 'password'];


{% endhighlight %}

The next time you start the suite without setting one of these values, an exception will be thrown. 

For optional parameters, you should set default values. The `$config` property is used to define optional parameters as well as their values. In the WebDriver module we use default Selenium Server address and port. 

{% highlight php %}

<?php
class WebDriver extends \Codeception\Module
{
    protected $requiredFields = ['browser', 'url'];    
    protected $config = ['host' => '127.0.0.1', 'port' => '4444'];
    

{% endhighlight %}

The host and port parameter can be redefined in the suite config. Values are set in the `modules:config` section of the configuration file.

{% highlight yaml %}

modules:
    enabled:
        - WebDriver:
            url: 'http://mysite.com/'
            browser: 'firefox'
        - Db:
            cleanup: false
            repopulate: false

{% endhighlight %}

Optional and mandatory parameters can be accessed through the `$config` property. Use `$this->config['parameter']` to get its value.

### Dynamic Configuration With Params

Module can dynamically be configured from environment variables. Parameter storage should be specified in global `codeception.yml` config inside `params` section. Parameters can be loaded from environment vars, from yaml (Symfony format), .env (Laravel format) or ini file. 

Use `params` section of global config `codeception.yml` to specify how to load them. You can specify several sources for params to be loaded from.

Example: load parameters from environment:

{% highlight yaml %}

params:
    - env # load params from environment vars

{% endhighlight %}

Example: load params yaml file (Symfony)

{% highlight yaml %}

params:
    - app/config/parameters.yml

{% endhighlight %}

Example: load params from env file (Laravel)

{% highlight yaml %}

params:
    - .env
    - .env.testing

{% endhighlight %}

Once loaded, param variables can be used as module configuration values. Use variable name wrapped with `%` as placeholder and it will be replaced with its value. 

Let's say we want to specify credentials for cloud testing service. We have loaded `SAUCE_USER` and `SAUCE_KEY` variables from environment, and now we are passing their values into config of `WebDriver`:

{% highlight yaml %}

    modules:
       enabled:
          - WebDriver:
             url: http://mysite.com
             host: '%SAUCE_USER%:%SAUCE_KEY%@ondemand.saucelabs.com'

{% endhighlight %}

Params are also useful to provide connection credentials for `Db` module (taken from .env files of Laravel).

{% highlight yaml %}

module:
    enabled:
        - Db:
            dsn: "mysql:host=%DB_HOST%;dbname=%DB_DATABASE%"
            user: "%DB_USERNAME%"
            password: "DB_PASSWORD"

{% endhighlight %}

### Runtime Configuration

If you want to reconfigure a module at runtime, you can use the `_reconfigure` method of the module.
You may call it from a helper class and pass in all the fields you want to change.

{% highlight php %}

<?php
$this->getModule('WebDriver')->_reconfigure(array('browser' => 'chrome'));


{% endhighlight %}

At the end of a test, all your changes will be rolled back to the original config values.

## Conclusion


Modules are the real power of Codeception. They are used to emulate multiple inheritances for Actor classes (UnitTester, FunctionalTester, AcceptanceTester, etc). Codeception provides modules to emulate web requests, access data, interact with popular PHP libraries, etc. If bundled modules are not enough for you that's ok, you are free to write your own! Use Helpers (custom modules) for everything that Codeception can't do out of the box. Helpers also can be used to extend the functionality of original modules.



* **Next Chapter: [ReusingTestCode >](/docs/06-ReusingTestCode)**
* **Previous Chapter: [< UnitTests](/docs/05-UnitTests)**