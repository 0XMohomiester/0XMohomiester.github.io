---
title: "CEREAL LOGGER | 247CTF"
date: 2024-06-16
categories: [CHALLENGES, CEREAL LOGGER]
tags: [CHALLENGES, CTF] 
---


Hi, in this writeup I will solve the new hard challenge `CEREAL LOGGER`, from [247ctf](https://247ctf.com/). I will also explain how I created a simple proxy server between `SQLmap` and the challenge server that can manipulate SQLmap requests.

## Challenge source code analysis
![IMG1](https://github.com/0XMohomiester/0XMohomiester.github.io/assets/47929033/24a5b14d-3451-4ac3-a4e2-5855c6ce6cfb)

1) class `insert_log` is defined and has a property `new_data` has constant value "Valid access logged!"
and has `__destruct()` methode that create a new SQLite3 database connection using a database file located at `/tmp/log.db` and finally, an SQL command is executed to insert a log message `$this->new_data` into the `log` table of the database.

2) Main Script Logic: 
  - qwef
  - wefwef 
