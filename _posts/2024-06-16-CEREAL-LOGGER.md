---
title: "CEREAL LOGGER | 247CTF"
date: 2024-06-16
categories: [CHALLENGES, CEREAL LOGGER]
tags: [CHALLENGES, CTF] 
---


Hi, in this writeup we will solve the new hard challenge `CEREAL LOGGER` from [247ctf](https://247ctf.com/)

```
<?php

  class insert_log
  {
      public $new_data = "Valid access logged!";
      public function __destruct()
      {
          $this->pdo = new SQLite3("/tmp/log.db");
          $this->pdo->exec("INSERT INTO log (message) VALUES ('".$this->new_data."');");
      }
  }

  if (isset($_COOKIE["247"]) && explode(".", $_COOKIE["247"])[1].rand(0, 247247247) == "0") {
      file_put_contents("/dev/null", unserialize(base64_decode(explode(".", $_COOKIE["247"])[0])));
  } else {
      echo highlight_file(__FILE__, true);
  }
```

