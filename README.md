# Attribute Utilities

[![Latest Version on Packagist][ico-version]][link-packagist]
[![Software License][ico-license]](LICENSE.md)
[![Total Downloads][ico-downloads]][link-downloads]

AttributeUtils provides utilities to simplify working with and reading Attributes in PHP 8.0 and later.

Its primary tool is the Class Analzyer, which allows you to analyze a given class (or enum, in PHP 8.1) with respect to some attribute class.  Attribute classes may implement various interfaces in order to opt-in to additional behavior, as described below.  The overall intent is to provide a simple but powerful framework for reading metadata off of a class, including with reflection data.

## Install

Via Composer

``` bash
$ composer require crell/attributeutils
```

## Usage

### Basic usage

The most important class in the system is `Analyzer`, which implements the `ClassAnalyzer` interface.  It has one optional dependency, `AttributeParser`.  For casual usage you do not need it, although if you are wiring `Analyzer` into a DI system you should inject it as its own service so that you can guarantee only one instances is created.

```php

#[MyAttribute(a: 1, b: 2)]
class Point
{
    public int $x;
    public int $y;
    public int $z;
}

$analyzer = new Crell\AttributeUtils\Analyzer();

$attrib = $analyzer->analyze(Point::class, MyAttribute::class);

// $attrib is now an instance of MyAttribute.
print $attrib->a . PHP_EOL; // Prints 1
print $attrib->b . PHP_EOL; // Prints 2
```

All interaction with the reflection system is abstracted away by the `Analyzer`.

You may analyze any class with respect to any attribute.  If the attribute is not found, a new instance of the attribute class will be created with no arguments, that is, using whatever it's default argument values are.  If any arguments are required, a `RequiredAttributeArgumentsMissing` exception will be thrown.

The net result is that you can analyze a class with respect to any attribute class you like, as long as it has no required arguments.

The most important part of `Analyzer`, though, is that it lets attributes opt-in to additional behavior to become a complete class analysis and reflection framework.

### Reflection

If a class attribute implements [`Crell\AttributeUtils\FromReflectionClass`](src/FromReflectionClass.php), then once the attribute has been instantiated the `ReflectionClass` representation of the class being analyzed will be passed to the `fromReflection()` method.  The attribute may then save whatever reflection information it needs, however it needs.  For example, if you want the attribute object to know the name of the class it came from, you can save `$reflection->getName()` and/or `$reflection->getShortName()` to non-constructor properties on the object.  Or, you can save them if and only if certain constructor arguments were not provided.

If you are saving a reflection value literally, it is *strongly recommended* that you use a property name consistent with those in the [`ReflectClass`](src/Attributes/Reflect/ReflectClass.php) attribute.  That way, the names are consistent across all attributes, even different libraries, and the resulting code is easier for other developers to read and understand.  (We'll cover `ReflectClass` more later.)

In the following example, an attribute accepts a `$name` argument.  If one is not provided, the class's short-name will be used instead.

```php
#[\Attribute]
class AttribWithName implements FromReflectionClass 
{
    public readonly string $name;
    
    public function __construct(?string $name = null) 
    {
        if ($name) {
            $this->name = $name;
        }
    }
    
    public function fromReflection(\ReflectionClass $subject): void
    {
        $this->name ??= $subject->getShortName();
    }
}
```

The reflection object itself should *never ever* be saved to the attribute object.  Reflection objects cannot be cached, so doing so would render the attribute object uncacheable.  It's also wasteful, as any data you need can be retrieved from the reflection object and saved individually.

There are similarly [`FromReflectionProperty`](src/FromReflectionProperty.php), [`FromReflectionMethod`](src/FromReflectionMethod.php), [`FromReflectionClassConstant`](src/FromReflectionClassConstant.php), and [`FromReflectionParameter`](src/FromReflectionParameter.php) interfaces that do the same for their respective bits of a class.

### Additional class components

The class attribute may also opt-in to analyzing various portions of the the class, such as its properties, methods, and constants.  It does so by implementing the [`ParseProperties`](src/ParseProperties.php), [`ParseMethods`](src/ParseMethods.php), or [`ParseClassConstants`](src/ParseClassConstants.php) interfaces, respectively.  They all work the same way, so we'll look at properties in particular.

An example is the easiest way to explain it:

```php
#[\Attribute(\Attribute::TARGET_CLASS)]
class MyClass implements ParseProperties
{
    public readonly array $properties;

    public function propertyAttribute(): string
    {
        return MyProperty::class;
    }

    public function setProperties(array $properties): void
    {
        $this->properties = $properties;
    }

    public function includePropertiesByDefault(): bool
    {
        return true;
    }
}

#[\Attribute(\Attribute::TARGET_PROPERTY)]
class MyProperty
{
    public function __construct(
        public readonly string $column = '',
    ) {}
}

#[MyClass]
class Something
{
    #[MyProperty(column: 'beep')]
    protected property $foo;
    
    private property $bar;
}

$attrib = $analyzer->analyze(Something::class, MyClass::class);
```

In this example, the `MyClass` attribute will first be instantiated. It has no arguments, which is fine.  However, the interface methods specify that the Analyzer should then parse `Something`'s properties with respect to `MyProperty`.  If a property has no such attribute, it should be included anyway and instantiated with no arguments.

The Analyzer will dutifully create an array of two `MyProperty` instances, one for `$foo` and one for `$bar`; the former having the `column` value `beep`, and the latter having the default empty string value.  That array will then be passed to `MyClass::setProperties()` for `MyClass` to save, or parse, or filter, or do whatever it wants.

If `includePropertiesByDefault()` returned `false`, then the array would have only one value, from `$foo`.  `$bar` would be ignored.

Note: The array that is passed to `setProperties` is indexed by the name of the property already, so you do not need to do so yourself.

The property-targeting attribute (`MyProperty`) may also implement `FromReflectionProperty` to get the corresponding `ReflectionProperty` passed to it, just as the class attribute can.

Be aware that reflection does not automatically differentiate between static and object properties.  By default, you will get both.  If you want to get only object properties (or only static), you can extract that value from reflection and then filter accordingly.  For example, like so:

```php
#[\Attribute(\Attribute::TARGET_PROPERTY)]
class MyProperty implements FromReflectionProperty
{
    public readonly bool $isStatic;

    public function __construct(
        public readonly string $column = '',
    ) {}
    
    public function fromReflection(\ReflectionProperty $subject): void
    {
        $this->isStatic = $subject->isStatic();
    }
}

#[\Attribute(\Attribute::TARGET_CLASS)]
class MyClass implements ParseProperties
{
    public readonly array $objectProperties;
    
    public readonly array $staticProperties;

    public function setProperties(array $properties): void
    {
        $this->objectProperties = array_filter(fn(MyProperty $p):bool => !$p->isStatic, $properties);
        $this->staticProperties = array_filter(fn(MyProperty $p):bool => $p->isStatic, $properties);
    }
    
    // ...
}
```

The `ParseClassConstant` interface works the same way as `ParseProperties`.

### Methods

`ParseMethods` works the same way as `ParseProperties` (including the caveat about static vs object methods).  However, a method-targeting attribute may also itself implement [`ParseParameters`](src/ParseParameters.php) in order to examine parameters on that method.  `ParseParameters` repeats the same pattern as `ParseProperties` above, with the methods suitably renamed.

### Excluding values

When parsing components of a class, whether they are included depends on a number of factors.  The `includePropertiesByDefault()`, `includeMethodsByDefault()`, etc. methods on the various `Parse*` interfaces determine whether components that lack an attribute should be included with a default value, or excluded entirely.

If the `include*()` method returns true, it is still possible to exclude a specific component if desired.  The attribute for that component may implement the [`Excludable`](src/Excludable.php) interface, with has a single method, `exclude()`.

What then happens is the Analyzer will load all attributes of that type, then filter out the ones that return `true` from that method.  That allows individual properties, methods, etc. to opt-out of being parsed.  You may use whatever logic you wish for `exclude()`, although the most common approach will be something like this:

```php
#[\Attribute(\Attribute::TARGET_PROPERTY)]
class MyProperty implements Excludable
{
    public function __construct(
        public readonly bool $exclude = false,
    ) {}
    
    public function exclude(): bool
    {
        return $this->exclude;
    }
}

class Something
{
    #[MyProperty(exclude: true)]
    private int $val;
}
```

If you are taking this manual approach, it is strongly recommended that you use the naming convention here for consistency.

### Attribute inheritance

By default, attributes in PHP are not inheritable.  That is, if class `A` has an attribute on it, and `B` extends `A`, then asking reflection what attributes `B` has will find none.  Sometimes that's OK, but other times it is highly annoying to have to repeat values.

`Analyzer` addresses that limitation by letting attributes opt-in to being inherited.  Any attribute &mdash; for a class, property, method, constant, or parameter &mdash; may also implement the [`Inheritable`](src/Inheritable.php) marker interface.  This interface has no methods, but signals to the system that it should itself check parent classes and interfaces for an attribute if it is not found.

For example:

```php
#[\Attribute(\Attribute::TARGET_CLASS)]
class MyClass implements Inheritable
{
    public function __construct(public string $name = '') {}
}

#[MyClass(name: 'Jorge')]
class A {}

class B extends A {}

$attrib = $analyzer->analyze(B::class, MyClass::class);

print $attrib->name . PHP_EOL; // prints Jorge
```

Because `MyClass` is inheritable, the Analyzer notes that it is absent on `B` so checks class `A` instead.  All attribute components may be inheritable if desired just by implementing the interface.

When checking for inherited attributes, ancestor classes are all checked first, then implementing interfaces, in the order returned by `class_implements()`.  Properties will not check for interfaces, of course, as interfaces cannot have properties.

### Attribute child classes

When checking for an attribute, the Analyzer uses an `instanceof` check in Reflection.  That means a child class, or even a class implementing an interface, of what you specify will still be found and included.  That is true for all attribute types.

### Sub-attributes

`Analyzer` can only parse a single attribute on each target.  However, it also supports the concept of "sub-attributes."  Sub-attributes work similarly to the way a class can opt-in to parsing properties or methods, but for sibling attributes instead of child components.  That way, any number of attributes on the same component can be folded together into a single attribute object.  Any attribute for any component may opt-in to sub-attributes by implementing [`HasAttributes`](src/HasSubAttributes.php).

The following example should make it clearer:

```php
#[\Attribute(\Attribute::TARGET_CLASS)]
class MainAttrib implements HasSubAttributes
{
    public readonly int $age;

    public function __construct(
        public readonly string name = 'none',
    ) {}

    public function subAttributes(): array
    {
        return [Age::class => 'fromAge'];
    }
    
    public function fromAge(?ClassSubAttribute $sub): void
    {
        $this->age = $sub?->age ?? 0;
    }
}

#[\Attribute(\Attribute::TARGET_CLASS)]
class Age
{
    public function __construct(public readonly int $age = 0) {}
}

#[MainAttrib(name: 'Larry')]
#[Age(21)]
class A {}

class B {}

$attribA = $analyzer->analyze(A::class, MainAttrib::class);

print "$attribA->name, $attribA->age\n"; // prints "Larry, 21"

$attribB = $analyzer->analyze(B::class, MainAttrib::class);

print "$attribB->name, $attribB->age\n"; // prints "none, 0"
```

The `subAttributes()` method returns an associative array of attribute class names mapped to methods to call.  After the `MainAttrib` is loaded, the Analyzer will look for any of the listed sub-attributes, and then pass their result to the corresponding method.  The main attribute can then save the whole sub-attribute, or pull pieces out of it to save, or whatever else it wants to do.

An attribute may have any number of sub-attributes it wishes.

Note that if the sub-attribute is missing, `null` will be passed to the method.  That is to allow a sub-attribute to have required parameters if and only if it is specified, while keeping the sub-attribute itself optional.  You therefore *must* make the callback method's argument nullable.

Sub-attributes may also be `Inheritable`.

### Multi-value attributes

By default, PHP attributes can only be placed on a given target once.  However, they may be marked as "repeatable," in which case multiple of the same attribute may be placed on the same target.  (Class, property, method, etc.)

The Analyzer does not support multi-value attributes, but it does support multi-value sub-attributes.  Specifically, if the sub-attribute is marked as multi-value, then an array of sub-attributes will be passed to the callback instead.

For example:

```php
#[\Attribute(\Attribute::TARGET_CLASS)]
class MainAttrib implements HasSubAttributes
{
    public readonly array $knows;

    public function __construct(
        public readonly string name = 'none',
    ) {}

    public function subAttributes(): array
    {
        return [Knows::class => 'fromKnows'];
    }
    
    public function fromKnows(array $knows): void
    {
        $this->knows = $knows;
    }
}

#[\Attribute(\Attribute::TARGET_CLASS)]
class Knows
{
    public function __construct(public readonly string $name) {}
}

#[MainAttrib(name: 'Larry')]
#[Knows('Kai')]
#[Knows('Molly')]
class A {}

class B {}
```

In this case, any number of `Knows` attributes may be included, but if included the `$name` argument is required.  The `fromKnows()` method will be called with a (possibly empty, in the case of `B`) array of `Knows` objects, and can do what it likes with it.  In this example the objects are saved in their entirety, but they could also be mushed into a single array or used to set some other value if desired.

Note that if a multi-value sub-attribute is `Inheritable`, ancestor classes will only be checked if there are no local sub-attributes.  If there is at least one, it will take precedence and the ancestors will be ignored.

### Caching

The main `Analyzer` class does no caching whatsoever.  However, it implements a `ClassAnalyzer` interface which allows it to be easily wrapped in other implementations that provide a caching layer.

For example, the [`MemoryCacheAnalyzer`](src/MemoryCacheAnalyzer.php) class provides a simple wrapper that caches results in a static variable in memory.  You should almost always use this wrapper for performance.

```php
$analyzer = new MemoryCacheAnalyzer(new Analyzer());
```

Other cache wrappers may also be implemented, and PSR-6 and PSR-16 wrappers are on the roadmap.  (PRs welcome in the meantime.)  Wrappers may also compose each other, so the following would be an entirely valid and probably good approach:

```php
$analyzer = new MemoryCacheAnalyzer(new Psr6CacheAnalyzer(new Analyzer()));
```

## Advanced features

There are a couple of other advanced features also available.  These are rarely useful, but if useful they can be very helpful.

### Transitivity

Transitivity applies only to attributes on properties, and only if the attribute in question can target both propeties and classes.  It is an alternate form of inheritance.  Specifically, if a property is typed to a class or interface, and the attribute in question implements `TransitiveProperty`, and the property does not have that attribute on it, then instead of looking up the inheritance tree the analyzer will first look at the class the property is typed for.

That's a lot of conditionals, so here's an example to make it clearer:

```php

#[\Attribute(\Attribute::TARGET_PROPERTY)]
class MyClass implements ParseProperties
{
    public readonly array $properties;

    public function setProperties(array $properties): void
    {
        $this->properties = $properties;
    }
    
    public function includePropertiesByDefault(): bool { return true; }

    public function propertyAttribute(): string { return FancyName::class; }
}


#[\Attribute(\Attribute::TARGET_PROPERTY | \Attribute::TARGET_CLASS)]
class FancyName implements Transitive
{
    public function __construct(public readonly string $name = '') {}
}

class Stuff
{
    #[FancyName('A happy little integer')]
    protected int $foo;

    protected string $bar;
    
    protected Person $personOne;
    
    #[FancyName('Her Majesty Queen Elizabeth II')]
    protected Person $personTwo;
}

#[FancyName('I am not an object, I am a free man!')]
class Person
{
}

$attrib = $analyzer->analyze(Stuff::class, MyClass::class);

print $attrib->properties['foo']->name . PHP_EOL; // prints "A happy little integer"
print $attrib->properties['bar']->name . PHP_EOL; // prints ""
print $attrib->properties['personOne']->name . PHP_EOL; // prints "I am not an object, I am a free man!"
print $attrib->properties['personTwo']->name . PHP_EOL; // prints "Her Majesty Queen Elizabeth II"
```

Because `$personTwo` has a `FancyName` attribute, it behaves as normal.  However, `$personOne` does not, so it jumps over to the `Person` class to look for the attribute and finds it there.

If an attribute implements both `Inheritable` and `Transitive`, then first the class being analyzed will be checked, then its ancestor classes, then its implemented interfaces, then the transitive class for which it is typed, and then that class's ancestors until it finds an appropriate attribute.

Both main attributes and sub-attributes may be declared `Transitive`.

### Custom analysis

As a last resort, an attribute may also implement the [`CustomAnalysis`](src/CustomAnalysis.php) interface.  If it does so, the analyzer itself will be passed to the `customAnalysis()` method of the attribute, which may then take whatever actions it wishes.  This feature is intended as a last resort only, and it's possible to create unpleasant infinite loops if you are not careful.  99% of the time you should use some other, any other mechanism.  But it's there if you need it.

### Dependency Injection

The Analyzer is designed to be usable on its own without any setup.  However, if configuring it as a service in a Dependency Injection framework it is strongly recommended that you make the `AttributeParser` its own service and inject it directly.  That ensures that only a single instance of that service will be created.  As it is stateless, there is no need to ever have more than one instance of it.

An appropriate cache wrapper should also be included in the DI configuration.

## Change log

Please see [CHANGELOG](CHANGELOG.md) for more information on what has changed recently.

## Testing

``` bash
$ composer test
```

## Contributing

Please see [CONTRIBUTING](CONTRIBUTING.md) and [CODE_OF_CONDUCT](CODE_OF_CONDUCT.md) for details.

## Security

If you discover any security related issues, please email larry at garfieldtech dot com instead of using the issue tracker.

## Credits

- [Larry Garfield][link-author]
- [All Contributors][link-contributors]

## License

The Lesser GPL version 3 or later. Please see [License File](LICENSE.md) for more information.

[ico-version]: https://img.shields.io/packagist/v/Crell/AttributeUtils.svg?style=flat-square
[ico-license]: https://img.shields.io/badge/License-LGPLv3-green.svg?style=flat-square
[ico-downloads]: https://img.shields.io/packagist/dt/Crell/AttributeUtils.svg?style=flat-square

[link-packagist]: https://packagist.org/packages/Crell/AttributeUtils
[link-scrutinizer]: https://scrutinizer-ci.com/g/Crell/AttributeUtils/code-structure
[link-code-quality]: https://scrutinizer-ci.com/g/Crell/AttributeUtils
[link-downloads]: https://packagist.org/packages/Crell/AttributeUtils
[link-author]: https://github.com/Crell
[link-contributors]: ../../contributors
