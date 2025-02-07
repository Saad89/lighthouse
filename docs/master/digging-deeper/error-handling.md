# Error Handling

Most of the error handling in Lighthouse is pretty closely based upon **webonyx/graphql-php**,
so you can find a lot of valuable information [in their documentation](https://webonyx.github.io/graphql-php/error-handling).

## User-friendly Errors

In a production setting, error messages should not be shown to the user by default
to prevent information leaking. In some cases however, you may want to display an
explicit error message to the user.

**webonyx/graphql-php** offers the [`GraphQL\Error\ClientAware`](https://github.com/webonyx/graphql-php/blob/master/src/Error/ClientAware.php) interface, that can
be implemented by Exceptions to control how they are rendered to the client.

Head over their [Error Handling docs](https://webonyx.github.io/graphql-php/error-handling) to learn more.

## Additional Error Information

The interface [`\Nuwave\Lighthouse\Exceptions\RendersErrorsExtensions`](https://github.com/nuwave/lighthouse/blob/master/src/Exceptions/RendersErrorsExtensions.php)
may be extended to add more information than just an error message to the rendered error output.

This custom exception contains information about the reason the exception was thrown:

```php
<?php

namespace App\Exceptions;

use Exception;
use Nuwave\Lighthouse\Exceptions\RendersErrorsExtensions;

class CustomException extends Exception implements RendersErrorsExtensions
{
    /**
    * @var @string
    */
    protected $reason;

    public function __construct(string $message, string $reason)
    {
        parent::__construct($message);

        $this->reason = $reason;
    }

    /**
     * Returns true when exception message is safe to be displayed to a client.
     */
    public function isClientSafe(): bool
    {
        return true;
    }

    /**
     * Returns string describing a category of the error.
     *
     * Value "graphql" is reserved for errors produced by query parsing or validation, do not use it.
     *
     * @api
     * @return string
     */
    public function getCategory(): string
    {
        return 'custom';
    }

    /**
     * Return the content that is put in the "extensions" part
     * of the returned error.
     *
     * @return array
     */
    public function extensionsContent(): array
    {
        return [
            'some' => 'additional information',
            'reason' => $this->reason,
        ];
    }
}
```

Now you can just throw that Exception somewhere in your code, for example your resolver,
and it will display additional error output.

```php
<?php

namespace App\GraphQL\Queries;

use App\Exceptions\CustomException;
use GraphQL\Type\Definition\ResolveInfo;
use Nuwave\Lighthouse\Support\Contracts\GraphQLContext;

class SomeField
{
    public function __invoke($rootValue, array $args, GraphQLContext $context, ResolveInfo $resolveInfo): string
    {
        if ($this->errorConditionIsMet()) {
            throw new CustomException(
                'This is the error message',
                'The reason why this error was thrown, is rendered in the extension output.'
            );
        }

        return 'Success!';
    }
}
```

A query that produces an error will render like this:

```json
{
  "data": null,
  "errors": [
    {
      "message": "This is the error message",
      "extensions": {
        "category": "custom",
        "some": "additional information",
        "reason": "The reason why this error was thrown, is rendered in the extension output."
      }
    }
  ]
}
```

## Registering Error Handlers

Error handlers receive the errors that occur during GraphQL execution.
They can be used to log, filter or format the errors.

Add them to your `lighthouse.php` config file, for example:

```php
'error_handlers' => [
    \Nuwave\Lighthouse\Execution\ExtensionErrorHandler::class,
    \App\GraphQL\MyErrorHandler::class,
],
```

An error handler class must implement [`\Nuwave\Lighthouse\Execution\ErrorHandler`](https://github.com/nuwave/lighthouse/blob/master/src/Execution/ErrorHandler.php)

```php
<?php

namespace App\GraphQL;

use Closure;
use GraphQL\Error\Error;
use Nuwave\Lighthouse\Execution\ErrorHandler;

class ExtensionErrorHandler implements ErrorHandler
{
    public function __invoke(?Error $error, Closure $next): ?array
    {
        // You can discard errors by returning null
        if ($this->shouldBeDiscarded($error)) {
            return null;
        }

        // Error handlers are instantiated once per query
        $this->errorCount++;

        // Keep the pipeline going, last step formats the error into an array
        return $next($error);
    }
}
```

## Partial Errors

As a GraphQL query may return a partial result, you may not always want to abort
execution immediately after an error occurred.

Use the [`ErrorPool`](https://github.com/nuwave/lighthouse/blob/master/src/Execution/ErrorPool.php)
when you want to collect multiple errors before returning a result.

```php
try {
    // Something that might fail but still allows for a partial result
} catch (\Throwable $error) {
    $errorPool = app(\Nuwave\Lighthouse\Execution\ErrorPool::class);
    $errorPool->record($error);
}

return $result;
```

## Testing Errors

See [PHPUnit - Testing Errors](../testing/phpunit.md#testing-errors).
