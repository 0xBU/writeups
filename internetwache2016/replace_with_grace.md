# Replace with Grace
*(web60, solved by 268)* 
> Regular expressions are pretty useful. Especially when you need to search and replace complex terms.
![](/blog/content/images/2016/02/replace.png)

It's a site that does simple search and replace on an input string.
For example:
```
Search: /cow/
Replace: cat
Input: cows are cute <3
-> cats are cute <3
```

My thought process for this challenge was then to find some sort of command injection. I was guessing that 
the site was using UNIX `sed`, by using pseudocode like:
```
cmd = 's' + <search> + <replace> + '/'
system("echo <input> | sed -i cmd")
```
A command injection in this case could be done by making `<replace>` something like `/;cat flag;`. But this 
wasn't working. I tried a few more avenues to get command injection in. Nothing worked. I was still 
convinced this was command injection into a UNIX shell, so I gave up for a bit and did other challenges.

Coming back to the challenge, after solving a few other web challenges, made me realize that this is 
probably just feeding strings into PHP's (since all the other web challenges are written in PHP) search and 
replace function. I'm not a PHP developer, but googling for PHP search and replace leads me to 
[preg\_replace](http://php.net/manual/en/function.preg-replace.php). Okay, it's probably this... doesn't 
seem bad, but PHP is notorious for being dangerous, so let's search up "[preg\_replace 
dangerous](https://bitquark.co.uk/blog/2013/07/23/the_unexpected_dangers_of_preg_replace)". Sure enough, 
`preg_replace` has a "bug/feature/wtf" where appending an `/e` to the `<search>` parameter will cause the 
`<replace>` parameter to be executed as code. *Simple.* 

Testing some payloads, there seems to be a list of blocked words, that's nice that they tell us this error 
message and let us know our payload is on the right track. The blocked word checker is case-sensitive, and 
funny enough, PHP is not.

```
Search: /cow/e
Replace: syStEm("cat flag.php")
Input: Hi Mom!
-> $FLAG = IW{R3Pl4c3_N0t_S4F3}
```

