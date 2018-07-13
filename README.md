# PHPUnit at Selesti with Laravel

## Index

- Setting up PHPUnit
    - Environment Configs
- Preparing your tests
    - Migrations
    - Using `setUp()` and `tearDown()` 
    - Authenticating Users
- Feature Tests
    - Mocks
    - API Integration
    - Classes
- Unit Tests
    - Private/Protected
- Writing Tests
    - Arrange
    - Act
    - Assert
- Running Tests
    - Refactoring

## Setting up PHPUnit

Laravel luckily comes with some boilerplate code to get things going for us, this includes:

- Bootstrapping your application
- A base `TestCase.php` to extend from
- Custom testing helpers for JSON/Auth etc

The `CreatesApplication` trait spins up a minimal copy of Laravel and allows you a PHP entry point to make global changes to your application state. One example could be simplifying the hashing method to speed up your tests.

```php
public function createApplication()
{
    $app = require __DIR__.'/../bootstrap/app.php';
    $app->make(Kernel::class)->bootstrap();
    
    // This makes the hashing less random, but speeds up all the
    // methods by keeping it more simple. It's test data after
    // all and will get wiped immediately!
    Hash::setRounds(4);

    return $app;
}
```

We also have access to a prebuilt `phpunit.xml` file which PHPUnit will automatically read when running.

Most of this can remain the same however it is usefull to note a few key areas.

One area you may wish to change is `stopOnFailure="false"` as if you have 100 tests and test 2 stops, it will continue to run the rest and take up time. You may want to set this to `true` to kill the process and let you fix - unless of course you want to see the full result!

By default Laravel gives you 2 types of tests `Feature` and `Unit` you can rename these, or create other folders and continue adding them to the `<testsuites>` property e.g 

```xml
<testsuite name="Selenium">
    <directory suffix="Test.php">./tests/Selenium</directory>
</testsuite>
```

Now you can house all your Selenium specific tests in `./tests/Selenium`

### Environment Configs

Arguably the most important part of the config is the `<php>` section which allows us to set environment variables.

This array of variables effectively overwrites anything in your `.env` file, 9 times out of 10 you'll need to add the following 2 lines

```xml
<env name="DB_CONNECTION" value="sqlite"/>
<env name="DB_DATABASE" value=":memory:"/>
```

What this will do is set the Laravel Database Driver to `sqlite` rather than hitting a real database and either nuking its data or slowing it down.

You then can set the database to a magic method called `:memory:` - What this will do is spin up a sqlite database in memory whilst the tests are running rather than writing to a real file - this again speeds up the testing process and keeps your real data intact.

> Remember; you are testing your **applications code**, so we don't need to worry about using a real database - thats Oracles job now!

## Preparing Your Tests

Laravel will give you 2 example tests to... delete, they are purely there for informative reasons! They are empty and located in `tests/Feature/ExampleTest.php` and `tests/Unit/ExampleTest.php`

You can create your own ones using the CLI - you have 2 options here, either to generate a Feature test or a Unit test (will explain differences later)

To create a unit test we can use `php artisan make:test UserServiceTest --unit`

To create a feature test we can use `php artisan make:test UserApiTest`

Simple!

This will create you a class looking something like

```php
<?php

namespace Tests\Unit;

use Tests\TestCase;
use Illuminate\Foundation\Testing\WithFaker;
use Illuminate\Foundation\Testing\RefreshDatabase;

class UserServiceTest extends TestCase
{
    /**
     * A basic test example.
     *
     * @return void
     */
    public function testExample()
    {
        $this->assertTrue(true);
    }
}

```

You'll most likely not need this, but you can edit if you want - the 2 things you want to look at are the 2 traits it imports allowing you to use if you want e.g 

```php
use Illuminate\Foundation\Testing\WithFaker;
use Illuminate\Foundation\Testing\RefreshDatabase;
```

### WithFaker

This imports a copy of faker into your tests so you can generate fake data e.g

```php
$fakeCompanyName = $this->faker()->companyName();
```

### RefreshDatabase

As each test is designed to run in isolation, so you do not want leaked data from 1 test into another, e.g

```php
function testOne()
{
    User::create():
    
    User::count(); // 1
}

function testTwo()
{
    User::create():
    
    User::count(); // 2
}
```

As you can see here you would be getting side effects if you needed to test a count or fetch data etc - to avoid things like this Laravel provides a fresh copy of your application for each test method.

If your tests do not require any database connectivity you can just ignore this trait, e.g if you're testing some mathmatic functions like `MathsService::addTogether(1, 2); // 3` you're fine!

However if you need database interations e.g you need to test relationships etc, then you need to import the `RefreshDatabase` trait - this will between each test effectively nuke your database and reconstruct it from your migration files (in what ever way it believes most performant) - it wil also run the default `DatabaseSeeder` to set up any default things you need.

> Remember; Only `use RefreshDatabase;` when needed as it adds performance overhead and slowly eats into the run time of tests. And if a test suite takes too long to run, people are less likely to continuously run it.

### Using `setUp()` and `tearDown()`

Sometimes you need to run repetative tasks before your tests, e.g imagine you're testing an API and you need to set up a user account, rather than doing it for every test, you can consolidate it into something like:

```php
class UserServiceTest extends TestCase
{
    /**
     * Creates a user with correct permissions
     */
    public function setUp()
    {
        $this->user = factory(User::class);
        $this->user->addresses()->save(new Address);
        $this->user->givePermissions('add news', 'edit news', 'delete news', 'list news');
    }
    
    public function testOne()
    {
        $this->user;
    }
    
    public function testTwo()
    {
        $this->user;
    }
}
```

Now before each test is run, you can have your user prepared for you without writing it a *thousand* times!

At the same time, we have a `tearDown` method, which you may have guessed.... runs after each test has run!

```php
    /**
     * Creates a user with correct permissions
     */
    public function setUp()
    {
        $this->user = factory(User::class);
        $this->user->givePermissions('upload photos');
    }

    public function test_i_can_upload_photos()
    {
        $this
            ->actingAs($this->user)
            ->post('upload-photo', [
                'photo' => FakeFile::class
            ]);
    }

    /**
     * Cleans up all the uploaded files so they don't use up your disk space!
     */
    public function tearDown()
    {
        Storage::deleteDirectory('./uploads/photos');
    }
```

### Authenticating Users

Another thing you might need to do when preparing your tests is to be authenticated. Most actions require the user to be logged in.

This is pretty easy in Laravel, typically we do it via 1 of 2 ways. 

#### Laravel Passport

If the application is API driven, then we use Passports facade to log us in for all API calls e.g.

```php
Passport::actingAs(
    factory(User::create)->create()
);
```

Now ever API call we make it will authenticate you as that user.

#### Session/Auth Driver

If you're just using a normal authentication flow by a form for example, then each call we make to our application we define the user we're doing it on the behalf of e.g.

```php
$this
    ->actingAs($this->user)
    ->post('upload-photo', [
        'photo' => FakeFile::class
    ]);
```

## Feature Tests

They try to keep terminology in Laravel simple, having on 3 categories of tests

- Feature tests
- Browser tests
- Unit tests

You could think of it as..

- If you're using Dusk or Selenium it's a Browser test
- If you're testing a single function e.g `MathService::addTogether(1, 3)` it's a unit test.
- If it's anything else, then it's a Feature test.

That means all things like integration testing, end-to-end testing etc - they're all considered as feature tests.

----

Typically Feature tests will not test a single thing - they will enter through a chain of other functions and is the only way to test Private/Protected methods.

An example could be.

```php
public function test_an_authenticated_user_can_upload_a_photo()
{
    $this
    ->actingAs($this->user)
    ->post('upload-photo', [
        'photo' => FakeFile::class
    ]);}

```

This would end up hitting the following chain of calls.

```php
class PhotoUploadController
{
    public function __invoke($request)
    {
        $response = UploadService::saveFromRequest($request);
        
        return $response;
    }
}

class UploadService
{
    public function saveFromRequest($request)
    {
        if (!UserService::canUpload($request->user)) {
            throw new Exception('User does not have permission to upload photos');
        }
        
        // Continue...
    }
}

class UserService
{
    public function canUpload($user)
    {
        return $this->can($user, 'upload photos');
    }
    
    private function can($user, $permission)
    {
        return DB::permissions($user->id, $permission)->count();
    }    
}
```

As you can see here, the `PhotoUploadController` receives the uploaded file, it then passes it off to the `UploadService` which checks the user has permission via the `UserService` which internally calls a private method `can()`.

This chain of calling would be considered as a Feature test as it has the ability to test the whole feature, rather than just the isolated method.

Because the `can()` function is private, it can only be invoked by the `UserService` which means you cannot write a Unit test for it. What you can do however is create a test which inherantly tests it is doing what it is expected via the above process, or create a test which is more isolated, perhaps a unit test for `UserService::canUpload()` which will in turn call the `can()` method proving if it works or not.

## Unit Tests

As mentioned previously, Unit tests can be used for testing things in much more isolation - traditionally they will be used to test a single thing. However we're not getting into definitions - this is more about keeping a simple to follow structure rather than symantics.

So we don't mind if you're unit tests actually do a couple of things, e.g. something like

```php
public function test_i_can_post_new_ideas()
{
    $user = factory(User::class);
    
    $idea = $user->createIdea([
        'idea' => 'We should all go to Hawaii'
    ]);
    
    $this->assertNotNull($idea);
    $this->assertEquals(1, Idea::count());
}

```
