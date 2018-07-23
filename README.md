# PHPUnit at Selesti with Laravel

## Index

- Setting up PHPUnit
    - Environment Configs
- Preparing your tests
    - Migrations
    - Using `setUp()` and `tearDown()` 
    - Authenticating Users
- Testing Concepts
    - Feature Tests
    - Browser Tests
    - Unit Tests
    - TDD
- Writing Tests
    - Arrange
    - Act
    - Assert
    - ~~Mocks~~
    - ~~API Integration~~
- ~~Running Tests~~
    - ~~Refactoring~~

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

## Testing Concepts

In Laravel try to keep terminology simple, having on 3 categories of tests

- Feature tests
- Browser tests
- Unit tests

You could think of it as..

- If you're using Dusk or Selenium it's a Browser test
- If you're testing a single function e.g `MathService::addTogether(1, 3)` it's a unit test.
- If it's anything else, then it's a Feature test.

That means all things like integration testing, end-to-end testing etc - they're all considered as feature tests.

### Feature Tests

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

### Unit Tests

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

Will end up doing something like


```php
class User 
{
    public function createIdea($data)
    {
        if (!$user->can('create ideas')) {
            throw new Exception('This user does not have permission to create ideas.');
        }

        $idea = new Idea($data);
        $this->ideas()->save($idea);

        if ($base64 = data_get($data, 'photo', null)) {
            $idea->attachPhoto($base64);
        }

        if ($categories = data_get($data, 'categories')) {
            $idea->categories()->associate(...$categories);
        }

        event(
            new IdeaCreated($idea)
        );

        IdeaService::rebuildStatisticCache();

        return $idea;
    }
}
```

You can see this method is actually doing 7 things

- Running its own validation
- Creating the idea entity
- Attaching a photo
- Associating categories
- Firing an event
- Rebuilding the cache
- Returning the new idea entity

So although this method does actually cross over into the realm of potentially being a Feature test - we believe its quite acceptable to organise this within your Unit folders when you deem appropriate.

However remember - you can only run this sort of test on `public` methods, `protected` and `private` methods can only be tested by going via a public method - so these might be considered as feature tests.

### Browser Tests

Laravel comes with a built in API for using headless Google Chrome - meaning you can run browser tests via PHPUnit using Laravel Dusk.

It is built upon the Facebook Webdriver and only works with Chrome currently - however you can configure it to use any other Selenium Server like Browserstack.

It lets you test more general purpose things like if a contact or registration form works as expected.

```php

$this->browse(function ($browser) use ($user) {
    
    $browser->maximize();

    $browser->loginAs($user)
            ->visit('/contact')
            ->waitFor('.form.loaded', 5)
            ->type('email', $user->email)
            ->type('message', 'this is my message')
            ->press('Send')
            ->waitFor('.form.success')
            ->assertPathIs('/thank-you')
            ->assertSee('Thank you for your message');
});
```

### Test Driven Development

This is a massive concept which can be covered in much better detail by https://course.testdrivenlaravel.com/

However in a nut-shell it is the concept of instead of using your browser or something like postman to keep hitting an endpoint or a page, you write the code you WANT to interact with, a term coined as "Programming by wishful thinking" - Effectively it is write the code you WISH you could use, even if it doesn't exist yet.

> Cavet - Only really works well for bespoke functionality that doesn't have to integrate too deeply into other systems without having to worry about things like Mocking etc.

This is a very fast, solid way to develop (assuming you're not tasked with building a GUI) as it:

- Removes the need to use a browser,
- You fulfill user story requirements as you go,
- A side-affect is self documenting your application code,
- Allows you to refactor / clean up code once you know it works,
- Leaves you with a suite of tests for the future!

The general concept runs along the line of:

- Write the test
- Run the test
- Fix the failure
- Run the test
- Fix the failure
- etc etc etc
- Feature finished

## Writing Tests

A general rule of thumb is to try and make your tests only test your application code, don't worry about testing framework or cms code - it will only waste time, and you can't fix it anyway. An example might be within Laravel, do not create tests for all your relationships - as you're just testing the relationship system in laravel works - you're other tests should inherantly prove they are working anyway.

### Test Names

These don't need to follow PSR naming, often people find using underscores make them easier to read, e.g `test_a_user_can_create_a_post` to some might read better than `TestAUserCanCreateAPost`.

You should try and be clear and concise about your test name, use full sentences to be as descriptive as possible, even if its long! If your tests are passing, then it won't be regually read anyway!

### Test Structure

A good way to organise your test code is to use a pattern known as AAA, this stands for

- Arrange
- Act
- Assert

### Arrange

So the arrange concept effecitvely suggests that you organise everything you need to do to get the test ready, this could look something like. Anything which you need to be sure of, you should always hard-code. e.g If you know you need the user to be `id = 1` then pass it in.

```php
function test_a_user_can_create_a_new_post()
{
    // Arrange
    $user = factory(User::class)->create([
        'id' => 1,
    ]);
    
    $user->givePermission('create posts');
}
```

### Act / Action

This is where you're likely to start actually integrating with your application and where the most complex logic could likely happen, this is where you will make API calls, run methods, actions etc

```php
function test_a_user_can_create_a_new_post()
{
    // Arrange
    $user = factory(User::class)->create([
        'id' => 1,
    ]);
    
    $user->givePermission('create posts');
    
    // Act
    $post = $user->createPost([
        'title' => 'amazing idea',
        'body' => 'We should totes solve world hunger.',
    ]);
}
```


### Assert

Technically the assertion phase is where things are proved to be right/wrong - however it might be that if an exception is thrown in your `createPost` it will also cause the test to fail.

You will need to carefuly decide what you're actually testing for - and make sure the assertion tries to keep to this, often is it tempting to throw in as many assertions as possible, but this will just slow down your application and can lead people to start chasing geese in a test when the actual error is elsewhere.

There are a variety of assertions which can be used, Laravel provides many custom assertions for certain things, however you can always rely on the core PHPUnit assertions, or create your own!

You can see a selection of assertions https://laravel.com/docs/5.6/http-tests#response-assertions but they are mentioned in a variety of places throughout the website.

Its worth remembering that the hardcoded value should always be the 1st param you pass into an assertion, this is because some methods require this format, so they keep it consistent.

e.g If you're testing 2 things match you should do something like:

```php
$this->assertEquals(5, $idea->count); // Good
$this->assertEquals($idea->count, 5); // Bad
```

A basic assertion for the above test could be something like.

```php
function test_a_user_can_create_a_new_post()
{
    // Arrange
    $user = factory(User::class)->create([
        'id' => 1,
    ]);
    
    $user->givePermission('create posts');
    
    // Act
    $post = $user->createPost([
        'title' => 'amazing idea',
        'body' => 'We should totes solve world hunger.',
    ]);
    
    // Assert
    
    // If we expect the method to return a Post, we need to check it is.
    $this->assertInstanceOf(Post::class, $post);
    
    // We should test that the post was actually saved to the database
    $this->assertTrue($post->exists);
    
    // And test it was created on behalf of our test user.
    $this->assertEquals(1, $post->user_id);
}
```

### Mocking

Sometimes you do not want the whole flow of your application to execute, e.g. if you want to hit an API endpoint, which then fires an event which sends a notification to every user, but you're only interested in testing the response, you're able to provide a fake version of a class which mirrors your real classes API. This is called Mocking! Under the hood it uses a library called Mockery.

An example using laravels built in Mocks would look something like this, assuming you were testing a notification system.

```php

public function test_order_notifications_send()
{
    // Arrange
    $order = new Order;
    Notification::fake();
    
    // Act
    $order->ship();
    
    // Assert
    Notification::assertSentTo($order->user, OrderShipped::class);
}

```

If you need to mock some of your own classes it is a little more verbose. By default your mocked class  will not have any functionality, you need to assign them basic behaviour. To get started you simply pass the class you want to mock into Mockery like so...

```php
$userService = Mockery::mock(UserService::class);
```

However if you were to run `$userService->find(1);` it will return `void` or throw `Exception` as it has not been told it needs to exist. If you want it to have functionality, then you can use the following basics.

```php
$userService = Mockery::mock(UserService::class);

$userService
    ->shouldReceive('create') // This actually means "should it happen to receive a `create` call"
    ->with(true) // and the param happens to be `true`
    ->andReturn(new Admin); // then return a new Admin
```

So now the rest of your code whenever it calls `$userService->create(true);` it will return a `new Admin` - the original method might have done a whole lot more, but you only need that part of the functionality to work.

Sometimes you want it to implement every method, but do nothing, you can either use `Mockery::mock(Admin::class)->shouldIgnoreMissing()` or the shortcut `Mockery::spy()`s this will just return `null` for any method calls.

```php
$faked = Mockery::spy(User::class);
$faked->helloWorld(); // null
$faked->goodbyeWorld(); // null
```
