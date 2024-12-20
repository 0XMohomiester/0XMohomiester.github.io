---
title: "Exploiting PHP Unserialize POP Chains | Rootme"
date: 2024-12-20
categories: [CHALLENGES, PHP - Unserialize Pop Chain]
tags: [CHALLENGES, CTF] 
---
## Introduction
[PHP - Unserialize Pop Chain](https://www.root-me.org/en/Challenges/Web-Server/PHP-Unserialize-Pop-Chain)  is a very nice web challenge designed to understand the concept of **magic method chaining** in the deserialization process.

As we know that [serialization](https://hazelcast.com/foundations/distributed-computing/serialization/) is the process of converting a data object or a combination of code into a series of bytes, preserving the object's state in an easily transmittable form. Deserialization is the reverse operation of serialization.

## What is the Magic Methods?
[Magic methods](https://www.php.net/manual/en/language.oop5.magic.php) are special methods which override PHP's default's action when certain actions are performed on an object. All methods names starting with __ are reserved by PHP.

One of the most popular magic methods is `__construct()` which is called when the object created. 

![IMG1](https://github.com/user-attachments/assets/5b91a228-bcdd-4b05-b29a-095970437807)

## What is POP chain? 
In normal scenario of exploiting insecure deserialization vulnerability, an attacker controls a serialized object that is passed into `unserialize()`. This will then allow her the opportunity to hijack the execution flow of the application, by controlling the values passed into magic methods.

The problem is: What if the declared magic methods does not contain any usefull code to perform my attack?
Now the exploitation of unsafe deserialization is useless, right?

Unfortunately, if the magic method themselves are not exploitable, an attacker could still control the execution flow using something called POP chains.

POP stands for `Property Oriented Programming` which means that the attacker can control all of the properties of the deserialized object. POP chains work by chaining code "gadgets" together to achieve the attacker’s ultimate goal. These "gadgets" are code snippets borrowed from the codebase that the attacker uses to further her goal.

## Challenge source code analysis

![IMG2](https://github.com/user-attachments/assets/64700220-7dbe-4aa9-a1fc-05d71edc0787)

The script defines two PHP classes which are `GetMessage` and `WakyWaky`.

1) GetMessage class:
  - Constructor `__construct`: Initializes the object with the `receive` parameter. If the value is "HelloBooooooy", the  script terminates with a message.
  - `__toString` Method: Returns the value of the receive property as a string.
  - Destructor `__destruct`: Validates the receive property. If it’s not "HelloBooooooy", the script terminates. Otherwise, if `$getflag` is true, it includes a `flag.php` file and reveals the flag.

2) WakyWaky class:
  - `__wakeup` Method: Executes when the object is unserialized, echoing the msg property.
  - `__toString` Method: Sets `$getflag` to true and creates a new GetMessage object, returning its receive value.

3) If a data parameter is provided via a POST request, it’s unserialized.

## Spotting the bugs








## Chaining of bugs
## Manuel Exploiting
## Automated Exploitation


