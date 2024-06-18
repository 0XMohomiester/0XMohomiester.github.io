---
title: "CEREAL LOGGER | 247CTF"
date: 2024-06-16
categories: [CHALLENGES, CEREAL LOGGER]
tags: [CHALLENGES, CTF] 
---

Hi, in this writeup I will solve the new hard challenge `CEREAL LOGGER`, from [247ctf](https://247ctf.com/). I will also explain how I created a simple proxy server between `SQLmap` and the challenge server that can manipulate SQLmap requests.

## Challenge source code analysis.
![IMG1](https://github.com/0XMohomiester/0XMohomiester.github.io/assets/47929033/24a5b14d-3451-4ac3-a4e2-5855c6ce6cfb)

1) class `insert_log` is defined and has a property `new_data` has constant value "Valid access logged!"
and has `__destruct()` methode that create a new SQLite3 database connection using a database file located at `/tmp/log.db` and finally, an SQL query is executed to insert a log message `$this->new_data` into the `log` table of the database.

2) Main Script Logic: 
  - The code check if the cookie named `247` is set for incoming HTTP requests.
  - Split the cookie value at the dot with `explode(".", $_COOKIE["247"])` and Check if the second part of the split value concatenated with a random number between 0 and 247247247 equals the string "0".
  - If the condition is met, it decodes the first part of the cookie value from base64, unserializes it, and writes the result to `/dev/null` (which essentially discards the output).

## Spotting the bugs.

- The condition `explode(".", $_COOKIE["247"])[1].rand(0, 247247247) == "0"` is vulnerable to [Type juggling](https://medium.com/swlh/php-type-juggling-vulnerabilities-3e28c4ed5c09) because it relies on PHP's type coercion, PHP uses loose comparison (==), which means it will attempt to coerce the types to make the comparison. For example, when comparing a string to a number, PHP will convert the string to a number.
- In the `__destruct()` method vulnerable to SQLi, the SQL query is constructed by directly concatenating the `new_data` property into the SQL statement, but what is `__destruct()` methode ? by asking google: 
![IMG2](https://github.com/0XMohomiester/0XMohomiester.github.io/assets/47929033/3fff93f5-7be6-48c0-ad49-09670eac2244)

- Simple Testing: 
![IMG3](https://github.com/0XMohomiester/0XMohomiester.github.io/assets/47929033/ddbe1977-a921-4bfd-8c11-d2325a71a75e)

- At this point, we can identify another vulnerability: `insecure deserialization` because the script decode tha first part of the cookie before dot and then unserialize it directly using `unserialize()` function.

## Chaning of bugs 

1) Exoloiting type juggling vulnerability to bypass the if condition.

2) Build a class with same name and properties (Attributes or Variables). 

3) Injecting SQL payload in `new_data` attribute before serializing object and then encode it to base64 encoded data.

## Manuel Exploiting.
