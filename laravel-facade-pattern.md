## How Facade & Manager pattern works in Laravel

All Laravel essential components (features) are accessible from anywhere in your application using a Facade class. For example when you need to hash a user password you use the Hash Facade like this:

```PHP
$hashed =  Hash::make('password');
```

And the nice thing about this is that you don't have to specify the hashing algorithm and provide it with all the parameters it may need, all this is done for you behind the scenes by Laravel.

**So the question is HOW??**

To understand how this actually works let's follow the code and see what it does.

If we inspect the code for the Hash class, it looks like this:

```PHP
class Hash extends Facade
{
    /**
     * Get the registered name of the component.
     *
     * @return string
     */
    protected static function getFacadeAccessor()
    {
        return 'hash';
    }
}
```

That's it, as you can see there is no `make` method or anything it is just one method 'getFacadeAccessor'. The magic happens inside the parent `Facade` class, inside the parent class we can see a PHP magic method `__callStatic`:

```PHP
public static function __callStatic($method, $args)
{
    $instance = static::getFacadeRoot();

    if (!$instance) {
        throw new RuntimeException('A facade root has not been set.');
    }

    return $instance->$method(...$args);
}
```

From this code, we can see that the `make` method is really called on the `$instance` variable, which we get from the `getFacadeRoot` method, so let's go into it:

```PHP
public static function getFacadeRoot()
{
    return static::resolveFacadeInstance(static::getFacadeAccessor());
}
```

This method simply resolves an **object** (remember this object) from the app service container using the key defined in the getFacadeAccessor method as we saw previously on the definition of the Hash class, and if you are questioning who registered this **object**; the answer is Laravel.

When the application starts, during the booting step Laravel auto registers all Facades on the app service container.

The question now is what is this **object**; it is an instance of the `Manager` class, not directly but one of its implementations in our case it is a `HashManager`, so what is a manager?

First, let's discuss the need for it before we explain it. Tasks like hashing, sending an email, or storing files can be handled in different ways, for hashing there are a lot of algorithms, for sending emails there are a lot of mail providers, for storing files there are a lot of options (local disk, S3, google drive, etc), and the problem with that is how to provide a good abstraction to hide the inner workings of these different ways and make it easy to switch between them when we need to.

This is what the Manager pattern was designed for, it handles the construction of the correct implementation based on the app configuration files and then passes the method call to that concrete object.

A manager class extends the base abstract `Manager` class and implements the interface specific to its tasks in this case the `Hasher` interface, which defines what a hasher class can do.

```PHP
class HashManager extends Manager implements Hasher
{
    // ....
}
```

And on the manager, we can finally find the `make` method we were looking for:

```PHP
public function make($value, array $options = [])
{
    return $this->driver()->make($value, $options);
}
```

As we can see it doesn't do too much, it calls the `$this->driver()` method, then calls make on the returned object, so let's take a look into the `$this->driver()` method.

> Now you know why they are called drivers on the config files

The code for `$this->driver()` looks like this (I added the comments to clarify the code):

```PHP
public function driver($driver = null)
{
    // get the right driver config
    $driver = $driver ?: $this->getDefaultDriver();

    // throw if no driver config is defined
    if (is_null($driver)) {
        throw new InvalidArgumentException(sprintf(
            'Unable to resolve NULL driver for [%s].', static::class
        ));
    }

    // if not created yet, create an instance of the driver passing the config
    if (! isset($this->drivers[$driver])) {
        $this->drivers[$driver] = $this->createDriver($driver);
    }

    // return it
    return $this->drivers[$driver];
}
```

So if we define the hash driver in our config like this:

```PHP
// hashing.php

'driver' => 'bcrypt',
```

the `$this->createDriver()` method will use the `createBcryptDriver` method from the `HashManager` class to create the driver, this method looks like this:

```PHP
public function createBcryptDriver()
{
    return new BcryptHasher($this->config->get('hashing.bcrypt') ?? []);
}
```

So finally we arrived at the object on which the `make` method is really called.

Finally, I would like to say that I learned all this just by reading and following the Laravel source code, I may missed something but nevertheless, I learned a lot, and you can too, I encourage you to do so.

I hope you get something useful from this, if so share it with others.
