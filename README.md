# PHP RFC: throw_on_error declare: promote warnings to Exceptions
  * Version: 0.1
  * Date: 2020-06-29
  * Author: David Négrier <d.negrier@thecodingmachine.com>, George Peter Banyard <girgias@php.net>
  * Status: Under Discussion
  * First Published at: http://wiki.php.net/rfc/throw_on_error


## Introduction
Currently, various failure states or errors in PHP are signalled to the developer by a Warning,
and a ``false`` or ``null`` return value.
In the PHP 7.x, series of PHP over 1100 functions exhibit this behaviour [1] and are documented in the PHP documentation.

For PHP 8.0, some of these failure states have been converted to ``TypeError``s and ``ValueError``s.
However, many warnings don't fall in any of those two categories, most notably PHP's I/O functions
(e.g. ``fopen()``).
The main issue is that Warning, unlike exceptions, do not break out of the execution context,
a behaviour one might want when dealing with I/O failure conditions.

## Motivation
```php
declare(strict_types=1);

$content = file_get_contents('foobar.log');

// If "foobar.log" does not exist, the "parseLogs" message will be passed false, leading to a TypeError.
parseLogs($content);

function parseLogs(string $content) { /* ... */ }
```

The code above triggers a ``TypeError`` that is misleading for the developer.
The error is located at the previous statement while trying to fetch the content of the file ``foobar.log``.
It could be that the file does not exist, PHP doesn't have the permission to access it, etc.

It is therefore the responsibility of the developer to check the return value of such called function,
and verify if it is ``false``.

```php
$content = @file_get_contents('foobar.json');
if ($content === false) {
    throw new FileNotFoundException('Could not load file foobar.json');
}
```

Relying on developers to properly handle those return values and warnings is an issue,
as forgetting to do it once can lead to bugs difficult to find.

## Historic reasons
PHP's current behaviour was put in place before PHP introduced exceptions in PHP 5.0.
If those functions and APIs were designed today they would certainly use exceptions.
As breaking the execution (by throwing an exception) allows the developer to handle the failure state,
by using a try/catch, if it can or let it bubble up until the application decides it can handle it.

## Work around
One can obtain this behaviour by using a custom error handler.
The following snippet from the ErrorException documentation (or an error handler like Whoops)
is often executed to promote these warnings to Exceptions:

```php
function exception_error_handler($severity, $message, $file, $line) {
    if (!(error_reporting() & $severity)) {
        // This error code is not included in error_reporting
        return;
    }
    throw new ErrorException($message, 0, $severity, $file, $line);
}
set_error_handler("exception_error_handler");
```

However, this approach has several issues:
  * This requires a global bootstrap to be executed, which may be extra overhead in some environments (ex. unit testing)
  * The error handler cannot be set in library code without affecting other libraries
  * Adversely, libraries don't know if a custom error handler is enabled or not,
  making it hard to write a predictable error handling code
  * Backtraces refer to the error handler function rather than the line at which the error occurred
  * This requires an extra function overhead for both initialization and error handling

## Proposal
This RFC proposes an opt-in declare statement, similar to strict_types, to override the default warning
behaviour by promoting it to an exception.

The new ``throw_on_error`` declare directive, takes a boolean integer (``1`` for true, ``0`` for false).

When enabled, functions which normally emit an ``E_WARNING`` return ``false``/``null`` will throw an exception instead.
And other warnings such as using an undefined variable or arithmetic with strings will also throw an exception instead.

### Interaction with error_reporting
When deciding whether to promote an error to an exception, ``throw_on_error`` ignores the ``error_reporting`` INI setting.
This is done to ensure consistent behaviour across environments.

For example, the following exception is caught:
```php
declare(throw_on_error=1);
ini_set("error_reporting", 0);
try {
  file_get_contents('not_found.txt');
} catch(ErrorException $e) {
  echo "Caught exception";
}
```

### Interaction with error suppression operator
As the error suppression operator ``@`` only affects PHP errors, it will have no effect on a promoted ErrorException,
as it currently has no effect on any exception.

For example, the error suppression operators in the following snippet does not have any effect:
```php
<?php
declare(throw_on_error=1);
function doTheThing(){
  try {
    $fh = @fopen("not_found.txt", "r");
  } catch(ErrorException $e) {
    echo "I have caught an error!";
  }
}
```

### JSON Extension
The JSON extension has since PHP 7.3 [2] the following flag ``JSON_THROW_ON_ERROR`` which emits a ``JsonException``
instead of returning false when a failure arises. The ``throw_on_error`` declare statement would enable this behaviour
even without the flag.

In other words:
```php
declare(throw_on_error=1);
var_dump(json_decode("{", false, 512));
```
and
```php
var_dump(json_decode("{", false, 512, JSON_THROW_ON_ERROR));
```
would both emit a ``JsonException``.

## Backward Incompatible Changes
As ``throw_on_error`` defaults to 0, there are no backward incompatibilities.
Additionally, using an undefined declare() is an error, thus using ``throw_on_error`` is not a BC break.

## Proposed PHP Version
Next minor version (8.0).

## RFC Impact
### To SAPIs
None.

### To Existing Extensions
None.

### To Opcache
None.

### Open Issues
  * Name of declare directive
  * Exception type to throw

## Unaffected PHP Functionality
Any error triggered during compile time continues to operate normally,
as the declare directive only affect runtime behaviour.

## Future Scope
Potential future improvements (not part of this RFC) include:
### Specific exceptions
If this RFC passes, a following RFC will be submitted proposing adding specific exceptions for each error triggered.

```php
<?php
try {
    $fh = @fopen("not_found.txt", "r");
} catch(FileNotFoundException $e) {
  echo 'File not found';
}
```

Adding more specific exceptions can be done in a separate RFC as this will not introduce breaking changes (those more specific exceptions will extend ''ErrorException'' and therefore any catch of ''ErrorException'' will still catch them).

However, there will be a big number of exception classes to add and before adding those classes, a decision will need to be taken regarding the namespace of those exceptions (see: https://wiki.php.net/rfc/namespaces-in-core). Indeed, introducing exceptions (like a ''FileNotFoundException'' which is not a reserved keyword) would certainly introduce conflicts in existing code. The usage of a namespace will probably be needed.


## Proposed Voting Choices
Per the Voting RFC, there would be a single Yes/No vote requiring a 2/3 majority.

An optional exit poll will be made available to explain a "No" vote:
  * would cause confusion for copy-pasted code
  * do not like name choices
  * need a new I/O API
  * should not be a declaration
  * other

## Patches and Tests
Prototype patch (partially complete): https://github.com/php/php-src/compare/master...Girgias:throw-on-error-declare

## References
[1] https://thecodingmachine.io/introducing-safe-php
[2] [[rfc:json_throw_on_error||PHP RFC: JSON_THROW_ON_ERROR]]
