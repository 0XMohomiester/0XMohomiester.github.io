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

## What is POP chain? 

POP stands for `Property Oriented Programming` which means that the attacker can control all of the properties of the deserialized object. POP chains work by chaining code "gadgets" together to achieve the attacker’s ultimate goal. These "gadgets" are code snippets borrowed from the codebase that the attacker uses to further her goal.

In normal scenario of exploiting insecure deserialization vulnerability, an attacker controls a serialized object that is passed into `unserialize()`. This will then allow her the opportunity to hijack the execution flow of the application, by controlling the values passed into magic methods.

The problem is: What if the declared magic methods does not contain any usefull code to perform my attack?
Now the exploitation of unsafe deserialization is useless, right?


