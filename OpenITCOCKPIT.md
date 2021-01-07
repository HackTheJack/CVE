# OpenITCOCKPIT writeup

## Tool explanation

**OpenITCOCKPIT v. 3.7.2** offers a bugged webshell to execute bash command after some controls implemented.

After the study of the tool, we found out that the main function that caused the bug is 

```php
public function execNagiosPlugin($command)
```

where **command** is the command that we want to execute.

The function filters first on special characters in the following way

```php
if (strpos($command, ';') || strpos($command, '&&') || strpos($command, '$') ||
        strpos($command, '|') || strpos($command, '`'))
```

if the command contains one or more of these special characters the function stop its execution and return a warning message.



A second filter is executed on each command that begins with '**./**' that will be deleted and parsing the command and its arguments.

```php
if (strpos($command, './') === 0)
    //Parse ./ away
    $_command = explode('./', $command);
    //remove spaces to get raw command name
    $_command = explode(' ', $_command[1], 2);
     if (!isset($_command[0]) || !in_array($_command[0], $plugins)) {
                $this->send("\e[0;31mERROR: Forbidden command!\e[0m\n");

                return false;
            }
```

The "else" branch of the if command executes the same control without parsing '**./**'.



If the command passes the checks then the following command is executed:

```php
$this->exec(escapeshellcmd("su " . Configure::read('nagios.user') . " -c '" . $command . "'"),
                                   ['cwd' => Configure::read('nagios.basepath') . Configure::read('nagios.libexec'),]);
```

## Bug explanation

The function **escapeshellcmd** presents the following explanation:

"Following characters are preceded by a backslash:
 &#;`|*?~<>^()[]{}$\, \x0A and \xFF. ' and " are escaped only if they are not paired.
 On Windows, all these characters plus % and ! are preceded by a caret (^)."



We can bypass and execute an arbitrary command exploiting this function.

The exploit is possible thanks to the implementation of **su** command in Ubuntu, if we pass to the webshell the following command we can pass all the check and execute a command that we have chosen

```bash
ls ' -c 'echo "ciao"
```

this is possible because we insert into the command a pair of ' character and a pair of " and the function escapeshellcmd will not escape them as described above.

The server where OpenITCOCKPIT runs will execute the command

```bash
su nagios -c 'ls ' -c 'echo "ciao"'
```

and the implementation of Ubuntu's **su** command let us to execute the second command instead of the fist.

This is a simple command, now we have to exploit an arbitrary command and then make priviledge escalation to take superuser permission.


