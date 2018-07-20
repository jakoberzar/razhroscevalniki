# Debugging Xdebug
## Building & installation
1. Download xdebug source from [GitHub](https://github.com/xdebug/xdebug).
2. Follow instructions from the [README](https://github.com/xdebug/xdebug/blob/master/README.rst). The path to the extension (to add in the ini file) is `zend_extension="<path>/xdebug/modules/xdebug.so"`, where <path> is the folder where you cloned xdebug.

## Running
You should add the same settings to enable xdebug and run it in your IDE like normally.

## Recompiling
With `./rebuild.sh` you can recompile php to use the modified code.

## Debugging
### Logging
You can use the `php_printf()` function like a normal printf function, but it will echo the output just like PHP (in HTML if you use php serve file.php)

### GDB
You can debug the xdebug debugger using gdb:
1. `cd <path where you cloned xdebug>`
2. Run gdb with one of the following commands:
    * `gdb php --args file.php.`
    * `gdb --args php file.php`
    *  `gdb /usr/bin/php --args file.php`.
3. Inside gdb, run the command `add-symbol-file ./modules/xdebug.so`
4. Set any breakpoints you would like to, like `break xdebug.c:2512`. When asked if you would like to set the breakpoint after loading shared libraries, confirm with `y`.
5. Run the command `run`.

You can also set this up with VS Code & the Microsoft C/C++ extension. Here is how to setup the `launch.json` file:
* Change `program` property to `"/usr/bin/php"`,
* Change `args` to `["file.php"]`
* Add an object to `setupCommands`:
```json
{
    "description": "Add xdebug library",
    "text": "add-symbol-file ./modules/xdebug.so",
    "ignoreFailures": true
}
```
And that should be it :)

### Issues
If you encounter any problems in these instructions, please contact me or create an issue in this repository.

Thanks!