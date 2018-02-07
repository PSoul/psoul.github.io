---
title: SQL注入环境
date: 2017-11-28 15:45:40
tags:
	- PHP
	- 代码审计
categories: 渗透
---

一个简单的SQL注入代码

<!-- more -->

CODE:

    <?php
    $con = mysql_connect("localhost","root","root");
    if (!$con)
      {
      die('Could not connect: ' . mysql_error());
      }
    mysql_select_db("test", $con);
    $id = $_REQUEST[ 'id' ];
    $query  = "SELECT * FROM admin WHERE username = $id ";
    $result = mysql_query($query);
    while($row = mysql_fetch_array($result))
      {
      echo $row['0'] . " " . $row['1'];
      echo "<br />";
      }
    echo "<br/>";
    echo $query;
    mysql_close($con);
    ?>

    


