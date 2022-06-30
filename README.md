# PhpUnit - Tips

> Some tips and tricks for phpunit

![Banner](./banner.svg)

## Enable step-by-step debugging in vscode

1. Make sure your `phpunit.xml` file didn't have the `processIsolation` set to `true`, if so, change the value to `false` *(there is no way to disable this flag using command line arguments)*,
2. Add a breakpoint in one of your test scenario and press <kbd>F5</kbd> to start the debugger in vscode,
3. On the command line, run `vendor/bin/phpunit --no-coverage --configuration .config/phpunit.xml tests/Unit/app/Console/Commands/CheckDelegationsTest.php`,
4. vscode will stop on the breakpoint and you'll be able to use the debugger.

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
