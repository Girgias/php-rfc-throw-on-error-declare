# Exception hierarchy to replace PHP errors (E_WARNING, etc.)

This is a design document which attempts to establish an exception
hierarchy for failure states/warnings which will be thrown when
the declare directive ``throw_on_error`` (To Be Confirmed) is enabled.

## Design choice
As there are numerous failure states which use PHP's internal error handling system we opt for the following design:
  * Interfaces MUST cover a broad area, e.g. network, file systems, I/O, encodings, etc.
  * Exceptions MUST be specific and deal with a single failure state,
  e.g. file not found, insufficient disk space, incompatible encodings, etc.
  * Exceptions MAY use different exception codes for additional specificity,
  e.g. an HTTP status code for an HTTP request.

## Naming specification
  * Interfaces MUST NOT use the suffix ``Interface``.
  * Interfaces SHOULD NOT use the suffix ``Exception``,
  but MAY resort to it if the interface name is otherwise ambiguous.
  * Exception classes MUST NOT use the suffix ``Exception``.

## Proposed hierarchy
### Interfaces
```php
interface IO extends Throwable {}
```
Deals with I/O failure states.

```php
interface FileSystem extends IO {}
```
I/O failures related to file systems.
  
### Exceptions
```php
class FileNotFound extends Exception implements FileSystem {}
```
When a requested file does not exist.

```php
class InsufficientPermissions extends Exception implements FileSystem {}
```
When the operation asked cannot be performed due to insufficient file permission.

