# Mess of Hash
*(web50, solved by 170)*  
> Students have developed a new admin login technique. I doubt that it's secure, but the hash isn't 
crackable. I don't know where the problem is...

This was an interesting challenge. On unzipping the challenge you're presented with a single README.txt.
```
All info I have is this:
<?php

$admin_user = "pr0_adm1n";
$admin_pw = clean_hash("0e408306536730731920197920342119");

function clean_hash($hash) {
    return preg_replace("/[^0-9a-f]/","",$hash);
}

function myhash($str) {
    return clean_hash(md5(md5($str) . "SALT"));
}
```

The website listed in the challenge description is simply a login screen. It's unlikely the challenge is 
meant to be an SQL injection, or XSS, or anything like that. Checking some basic directories/files 
consistent with the other challenges tells me that `admin.php` doesn't exist, nor does `admin`, but 
`flag.php` is there, it's just not readable (`flag.php` page loads, it's just blank). 

So we have to somehow read that `flag.php`. There's really two ways, it's either just presented to us if we 
login as the 'pr0\_adm1n' user, or via SQLi. Let's see what we can think of for how to login as 
'pr0\_adm1n' knowing we have what seems to be the server-side hashing code, and the admin's hashed 
password. 

Path of least resistance... we can guess the hash is md5, based on the length of it, let's see if it's 
cracked already online! *Nope, google shows up nothing.* Okay, so the hashing code seems to take in a 
`$str`, which I guess is probably the password when a user is created in this fictional school (from the 
challenge description). The password is hashed, then the string "SALT" is appended to it, i.e. salting it, 
and then it's hashed again. And then for some reason, that I'm not too sure about, it seems to remove all 
non-hex characters. 

The hashing seems clean. I don't see how to get a hash collision, and bruteforce would be lame, and 
considering it's not found online, it's probably not a short password. I'm out of ideas here, the best I 
can think of is that this is another one of PHP's infamous security holes, and the `md5` function is 
somehow poorly implemented. Googling for "md5 php dangerous" leads me to [PHP: md5('240610708') == 
md5('QNKCDZO')](https://news.ycombinator.com/item?id=9484757). Interesting... so reading that it seems that 
PHP has a weird type issue:

> All of them start with 0e, which makes me think that they're being parsed as floats and getting converted 
to 0.0. 

This is exactly what we have. We just have to find a string that `myhash()`'s into something that starts 
with `0e`, and then it will collide with the check on `$admin_pw`. Turns out the requirement is a bit 
stricter, and every hex character after `0e` also has to be decimal, 0-9, for the conversion to float 0.0 
to happen. But, this is still doable, and we now have a much smaller search space.
```
<?php
$admin_user = "pr0_adm1n";
$admin_pw = clean_hash("0e408306536730731920197920342119");

function clean_hash($hash) {
    return preg_replace("/[^0-9a-f]/","",$hash);
}

function myhash($str) {
    return clean_hash(md5(md5($str) . "SALT"));
}

function generateRandomString($length = 10) {
    $characters = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
    $charactersLength = strlen($characters);
    $randomString = '';
    for ($i = 0; $i < $length; $i++) {
        $randomString .= $characters[rand(0, $charactersLength - 1)];
    }
    return $randomString;
}

for ($i = 0; $i < 100000000; $i++) {
	$result = generateRandomString(8);
	if (myhash($result) == $admin_pw) {
		print("woo!");
		print($result . "  " . myhash($result) . "\n");
	}
}
?>
```

```
woo!KyJg0SXC  0e588095185108872523046880371953
```

Logging in with the username, `pr0_adm1n`, and password, `KyJg0SXC`, will lead to a hash collision, and at 
login `flag.php` is displayed.

```
IW{T4K3_C4RE_AND_C0MP4R3}
```


