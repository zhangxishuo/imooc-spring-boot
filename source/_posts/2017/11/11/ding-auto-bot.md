---
title: 钉钉机器人自动推送
date: 2017-11-11 19:37:00
categories: 智能
tags:
- 钉钉
- php
---
[课表管理系统](https://github.com/yunzhiclub/courseManageSystem)，是我们用`php`开发的第一个`web`项目，虽然功能实现上没有太大的问题，但是代码的规范性还是不敢恭维。这个项目或许就这样永远停留在`Github`中，就这样，不去修改，这会警示着我们代码的整洁。

该项目已经上线，在不考虑代码规范性的前提下，系统的功能还是没有问题的。

![](/images/2017/11/11/ding-auto-bot/0.png)

这是显示的首页，每节课我们都需要去亲自登录系统去查看每个人的课程状态，过于麻烦。

正好，钉钉群中每天为我们推送消息的`Github`机器人给了我们灵感，我们可不可以自定义一个自动推送消息的机器人放在钉钉中呢？

<!-- more -->

# Demo

## 原理

钉钉的开放平台写的很详细，[钉钉|开放平台](https://open-doc.dingtalk.com/docs/doc.htm?spm=a219a.7629140.0.0.zZIvnt&treeId=257&articleId=105735&docType=1)

读一遍官方的`Demo`，感觉对整个流程有一个非常清晰的认识。

![](/images/2017/11/11/ding-auto-bot/1.png)

每个绑定到钉钉群的机器人都有一个`"Hook"`地址，其实就是我们每天都能见到的`url`，我们在这个地址上`post`指定格式的数据，就能让该机器人在钉钉群中发消息。

## 思路

在钉钉群中建立群机器人，获取`"Hook"`地址。

建立相应控制器，相应方法，该方法负责完成一系列获取数据、且向相应`"Hook"`地址`post`数据的操作。

建立脚本，该脚本访问映射到我们上文方法的特定`url`。

设置定时任务，定时执行该脚本，完成自动推送。

# 实现

## 添加机器人

添加机器人，选择自定义机器人。

![](/images/2017/11/11/ding-auto-bot/2.png)

![](/images/2017/11/11/ding-auto-bot/3.png)

这里我们获取到该机器人的`"Hook"`地址，我们将其记下，后面配置时会用到。

## 配置Hook

为了让我们的系统更加灵活，我们将机器人的`"Hook"`地址配置在`config.php`中，动态获取，方便以后修改。

在`application/config.php`中，添加一个`hook`配置项：

```php
// 钉钉机器人Hook地址
'hook' => 'https://oapi.dingtalk.com/robot/send?xxxxxx'
```

不要觉得这个配置文件有多复杂，其实言其本质，不过是一个`php`数组，这个数组可以在全局使用。

我们想添加一个配置项，不过是在该数组中添加一个元素。

## M层

我们先写`M`层的方法，官方`Demo`中给了一个`request_by_curl`方法，既然有现成的，我们直接拿过来用。

该方法参数一`$remote_server`为机器人的`"Hook"`地址，参数二`$post_string`为我们要推送的格式化数据。

```php
/**
 * 官方提供的推送方法
 */
public function request_by_curl($remote_server, $post_string) {

    $ch = curl_init();  
    curl_setopt($ch, CURLOPT_URL, $remote_server);
    curl_setopt($ch, CURLOPT_POST, 1); 
    curl_setopt($ch, CURLOPT_CONNECTTIMEOUT, 5); 
    curl_setopt($ch, CURLOPT_HTTPHEADER, array ('Content-Type: application/json;charset=utf-8'));
    curl_setopt($ch, CURLOPT_POSTFIELDS, $post_string);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);

    $data = curl_exec($ch);
    curl_close($ch);  
               
    return $data;  
}
```

该方法需要`"Hook"`地址和推送的数据，`"Hook"`地址已经配置好了，接下来我们写方法获取数据。

这里获取数据的逻辑比较复杂，我们将该方法拆分。

> 每个方法，只做一件事！————《代码整洁之道》

初次完成时，`getMessage`方法非常臃肿，这里，我将重构后的代码介绍一下。

`getState`方法用于获取用户在该节课的状态，`putMessage`方法调用`getState`，获取所有正常用户的信息，用`dataFormat`格式化数据，并拼接为数组返回。

`getMessage`方法调用`putMessage`，根据不同的时间节点调用方法，获取信息数组，同时在最后连接为字符串返回。

**注：这里用于连接的`dingMsg`一定是双引号字符串，`php`不会处理单引号字符串，只会替换双引号字符串中的特殊字符(如换行符等)。**

```php
/**
* 根据时间获取相应消息
*/
public function getMessage($timeMsg) {

    $knobs = [
        '[第一节]',
        '[第二节]',
        '[第三节]',
        '[第四节]',
        '[第五节]'
    ];

    $users = User::getUsualUsers();
    $term  = Term::getCurrentTerm();
    $week  = new Week();
    $time  = strtotime($term->start_time);

    $current_day  = User::getDay();
    $current_week = $week->WeekDay($time, time());

    foreach ($users as $key => $user) {
        $user->term = $term->id;
        $user->day  = $current_day;
    }

    $messages = [];

    if ($timeMsg == 'morning') {

        $messages = $this->putMessage($messages, $knobs, $users, $current_week, 1);
        $messages = $this->putMessage($messages, $knobs, $users, $current_week, 2);
    } else if ($timeMsg == 'afternoon') {

        $messages = $this->putMessage($messages, $knobs, $users, $current_week, 3);
        $messages = $this->putMessage($messages, $knobs, $users, $current_week, 4);
    } else if ($timeMsg == 'night') {

        $messages = $this->putMessage($messages, $knobs, $users, $current_week, 5);
    }

    $dingMsg = "";

    foreach ($messages as $key => $message) {

        $dingMsg = $dingMsg . $message . "\n";
    }

    return $dingMsg;
}

/**
* 拼接用户课程状态字符串
*/
public function putMessage($messages, $knobs, $users, $week, $knob) {

    array_push($messages, $knobs[$knob - 1]);

    foreach ($users as $key => $user) {
        
        $message = $this->getState($user, $week, $knob);
        $message = $this->dataFormat($key, $user, $message);

        array_push($messages, $message);
    }

    return $messages;
}

/**
* 获取用户状态信息
*/
public function getState($user, $week, $knob) {

    $user->knob = $knob;

    $state = $user->CheckedState($week);

    switch ($state) {
        case 1:
            $message = '请假';
            break;
        
        case 2:
            $message = '有课';
            break;

        case 3:
            $message = '加班';
            break;

        case 4:
            $message = '休息';
            break;

        case 5:
            $message = '无课';
            break;

        case 6:
            $message = '缺班';
            break;
    }

    return $message;
}

/**
* 格式化数据
*/
public function dataFormat($key, $user, $message) {

    $temp = '' . $key + 1 . '.' . $user->name . ' ' . $message;
    return $temp;
}
```

官方的`request_by_curl`需要接收特定格式的`Json`数据，所以我们还需要将`request_by_curl`封装一下，同时完成数据格式转换与数据推送。

```php
/**
 * 钉钉Hook推送消息方法
 */
public function autoPush($message) {

    $webhook = config('hook');

    $data = array (
        'msgtype'  => 'text',
        'text'     => array (
            'content' => $message
        )
    );

    $data_string = json_encode($data);

    $result = $this->request_by_curl($webhook, $data_string);

    echo $result;
}
```

## 控制器