# PhpUnit - Tips

> Some tips and tricks for phpunit

![Banner](./banner.svg)

- [Enable step-by-step debugging in vscode](#enable-step-by-step-debugging-in-vscode)
- [Output a text on the console when phpunit is started with --debug argument](#output-a-text-on-the-console-when-phpunit-is-started-with---debug-argument)
- [Improve performance tips](#improve-performance-tips)
  - [Avoid to use XDEBUG for code coverage but use phpcov](#avoid-to-use-xdebug-for-code-coverage-but-use-phpcov)
  - [Process isolation](#process-isolation)
    - [Cannot declare class ... because the name is already in use](#cannot-declare-class--because-the-name-is-already-in-use)
  - [BCRYPT](#bcrypt)

## Enable step-by-step debugging in vscode

1. Make sure your `phpunit.xml` file didn't have the `processIsolation` set to `true`, if so, change the value to `false` *(there is no way to disable this flag using command line arguments)*,
2. Add a breakpoint in one of your test scenario and press <kbd>F5</kbd> to start the debugger in vscode,
3. On the command line, run `vendor/bin/phpunit --stop-on-failure --no-coverage --configuration .config/phpunit.xml tests/Unit/app/Core`,
4. vscode will stop on the breakpoint and you'll be able to use the debugger.

`--stop-on-failure` is not strictly required but recommended during local development to not wait until all the tests suite is finished.

Under Laravel 8+, you can also use `php artisan test --stop-on-failure  --no-coverage --configuration .config/phpunit.xml tests/Unit/app/Core` for a better user experience.

## Output a text on the console when phpunit is started with --debug argument

> [https://stackoverflow.com/a/12612733/1065340](https://stackoverflow.com/a/12612733/1065340)

It seems there is no PHPUnit internal API to detect this but this workaround can be used:

```php
if (in_array('--debug', $_SERVER['argv'], true)) fwrite(STDOUT, __METHOD__ . ' - Use '. $fileName);
```

So, in your Test parent class (or just create a `Trait`), just add the following function:

```php
/**
 * Write a message to the console when phpunit is started with the `--debug` CLI argument
 *
 * @param string $message The message to write on the console
 *
 * @return void
 */
protected function debug(string $message): void {
    if (in_array('--debug', $_SERVER['argv'], true)) {
        $functionName=debug_backtrace()[1]['class'].'::'.debug_backtrace()[1]['function'];
        fwrite(STDOUT, '[DEBUG] '. $functionName . ' - ' .$message . "\n");
    }
}
```

And, from now, to use it write something like `$this->debug('Using input file: '. $path);`)

## Improve performance tips

### Avoid to use XDEBUG for code coverage but use phpcov

> [40 times faster PHP Code Coverage Reporting with PCOV](https://www.kurmis.com/2020/01/15/pcov-for-faster-code-coverage.html)

Install the pcov PHP extension and stop using XDEBUG (`XDEBUG_MODE=off`) will increase performance significantly.

```dockerfile
RUN set -e -x \
    echo "Install pcov"; \
    pecl install pcov && docker-php-ext-enable pcov; \
    apt-get clean; rm -rf /tmp/pear/* /tmp/* /var/list/apt/*
```

Then, just run phpunit and display the generated code coverage report:

```bash
XDEBUG_MODE=off vendor/bin/phpunit configuration=.config/phpunit.xml --coverage-text=.output/code-coverage.txt
head -n 10 .output/code-coverage.txt
```

Your `phpunit.xml` file can be defined like below to define where to put the code coverage cache information. Nothing more is needed.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit  ...>
    ...
    <coverage cacheDirectory=".output/code-coverage-cache" processUncoveredFiles="true">
        <include>
            <directory suffix=".php">./../app</directory>
        </include>
    </coverage>
    ...
</phpunit>
```

### Process isolation

Running phpunit with process isolation means that every tests will be fired in a "refreshed" environment and this is time consuming.

The ideal world is to be able to set `processIsolation="false"` in your `phpunit.xml` file like this:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit ... processIsolation="false" ...>
    ...
</phpunit>
```

Be careful with `process_isolation=false` since this can conduct to unpredictable behaviours.

The solution is to edit that class and add these two doc-block notations for the class itself:

```php
/**
 * @runInSeparateProcess
 * @preserveGlobalState disabled
 */
class MyTestTest extends PHPUnit\Framework\TestCase
```

We can also run that particular in the CLI using the `--process-isolation` argument.

#### Cannot declare class ... because the name is already in use

When your tests is using a mock-up like, for Laravel, `Mockery::mock`, you'll get a *Cannot declare class* error **as soon as** you'll ask code coverage. This because the code coverage will load every classes of your project and will detect if the class is covered or not.

When a class has been already *mocked-up*, an instance already exists in memory and thus, we'll get the error.

```text
PHP Fatal error:  Cannot declare class ..., because the name is already in use in ... on line ...
```

### BCRYPT

During tests, we can set `BCRYPT` to just 1 i.e. we don't need very secure passwords (crypted more than once); it's just for tests purposes so set `BCRYPT` to the smallest number: `1`.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit ... processIsolation="false" ...>
    ...
    <php>
        <env name="BCRYPT_ROUNDS" value="1" />
    </php>
</phpunit>
```
