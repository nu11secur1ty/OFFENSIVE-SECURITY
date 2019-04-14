***Custom Scripting***

Now that we have a feel for how to use **irb** to test API calls, let’s look at what objects are returned and test basic constructs. Now, no first script would be complete without the standard ‘Hello World’, so let’s create a script named ***helloworld.rb*** and save it to ***/usr/share/metasploit-framework/scripts/meterpreter.***

```bash
root@kali:~# echo “print_status(“Hello World”)” > /usr/share/metasploit-framework/scripts/meterpreter/helloworld.rb
```
We now execute our script from the console by using the ***run*** command.

```bash
meterpreter > run helloworld
[*] Hello World
meterpreter >
```


