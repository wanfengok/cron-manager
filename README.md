# cronManager

# 简介

cronManager是一个纯PHP实现的定时任务管理工具,api简单清晰,采用的是多进程模型,进程通信采用的是消息队列,任务监控也提供了简单的命令,方便易用

# 特性

* 多进程模型

* 支持守护进程

* 平滑重启

* 提供各种命令监控任务运行状态

* 支持任务分片,也就是多个进程分割运行一个任务

## 环境要求

1. liunx
2. pcntl扩展开启
3. php 5.4以上
4. composer


* 手动安装记得要先 `composer update`一下哦！

## 使用介绍

核心方法 `CronManager::taskInterval($name, $command, $callable, $ticks = [])` 

参数1 $name 定时任务名称

参数2 $command 

传入string则表示用key@value的形式表示
1. `s@n` 表示每n秒运行一次 
2. `i@n` 表示每n分钟运行一次 
3. `h@n` 表示每n小时运行一次
4. `at@nn:nn` 表示指定每天的nn:nn执行 例如每天凌晨 at@00:00

传入array则表示依次运行数组里的每一个指定日期,要求每一个元素都可以被`strtotime`函数解析,否则运行不了
如: `['2017-09-09 08:00','2017-09-09 08:00']`

参数3 $callable 回调函数,也就是定时任务业务逻辑

参数4 $ticks 用于任务分片

## 快速入门示例

``` php

//test.php

require __DIR__ . '/../vendor/autoload.php';


$manager = new SuperCronManager\CronManager();

// 设置worker数
$manager->workerNum = 5;

// 设置输出重定向,守护进程模式才生效
$manager->output = './test.log';

manager->taskInterval('每秒钟运行一次', 's@1', function(){
	echo "Hello crontabManager\n";
});
$manager->taskInterval('每分钟运行一次', 'i@1', function(){
	echo "Hello crontabManager\n";
});
$manager->taskInterval('每小时运行一次', 'h@1', function(){
	echo "Hello crontabManager\n";
});
$manager->taskInterval('每天凌晨运行一次', 'at@00:00', function(){
	echo "Hello crontabManager\n";
});
$manager->taskInterval('任务分片', 's@1', function($str){
	echo "$str\n";
},[1,2]);

$manager->taskInterval('指定日期运行', ['2017-12-20 23:06','2017-12-20 23:07'], function($index){
	echo "ticks $index\n";
});

$manager->run();


```

## 命令使用示例

* 检测扩展是否启动，缺少扩展将无法使用。 建议第一次运行先使用此命令查看扩展情况

> php test.php check

```
+----------+--------+------+------+
| name     | status | desc | help |
+----------+--------+------+------+
| php>=5.4 | [OK]   |      |      |
| pcntl    | [OK]   |      |      |
| posix    | [OK]   |      |      |
| sysvmsg  | [OK]   |      |      |
| sysvsem  | [OK]   |      |      |
| sysvshm  | [OK]   |      |      |
+----------+--------+------+------+
```

* 启动

>  php test.php

```
+------------+---------------------+
| pid        | 19629               |
+------------+---------------------+
| output     | /dev/null           |
+------------+---------------------+
| task_num   | 7                   |
+------------+---------------------+
| worker_num | 5                   |
+------------+---------------------+
| start_time | 2017-12-09 13:59:15 |
+------------+---------------------+

```

* 以守护进程方式启动（`无任何提示表示成功`）

>  php test.php -d

* 查看任务状态

>  php test.php status

```
+------------+---------------------+
| pid        |19690                |
+------------+---------------------+
| output     | ./test.log          |
+------------+---------------------+
| task_num   | 4                   |
+------------+---------------------+
| worker_num | 5                   |
+------------+---------------------+
| start_time | 2017-12-09 14:03:44 |
+------------+---------------------+
+----+-----------------------+----------+--------+-------+---------------------+---------------------+
| id | name                  | tag      | status | count | last_time           | next_time           |
+----+-----------------------+----------+--------+-------+---------------------+---------------------+
| 0  | 每秒钟运行一次        | s@1      | 正常   | 7     | 2017-12-09 14:03:51 | 2017-12-09 14:03:52 |
| 1  | 每分钟运行一次        | i@1      | 正常   | 0     | -                   | 2017-12-09 14:04:44 |
| 2  | 每小时运行一次        | h@1      | 正常   | 0     | -                   | 2017-12-09 15:03:44 |
| 3  | 指定每天00:00运行一次 | at@00:00 | 正常   | 0     | -                   | 2017-12-10 00:00:00 |
+----+-----------------------+----------+--------+-------+---------------------+---------------------+
```

* 查看worker状态

>  php test.php worker

```

+------------+---------------------+
| pid        | 19690               |
+------------+---------------------+
| output     | ./test.log          |
+------------+---------------------+
| task_num   | 4                   |
+------------+---------------------+
| worker_num | 5                   |
+------------+---------------------+
| start_time | 2017-12-09 14:03:44 |
+------------+---------------------+
+-------+------------+----------+---------------------+
| pid   | exec_count | memory   | start_time          |
+-------+------------+----------+---------------------+
| 19691 | 16         | 0.57(mb) | 2017-12-09 14:03:44 |
| 19692 | 19         | 0.57(mb) | 2017-12-09 14:03:44 |
| 19693 | 12         | 0.57(mb) | 2017-12-09 14:03:44 |
| 19694 | 12         | 0.57(mb) | 2017-12-09 14:03:44 |
| 19695 | 5          | 0.57(mb) | 2017-12-09 14:03:44 |
+-------+------------+----------+---------------------+

```
* 查看服务日志(这个有点LOW，就是直接读日志文件输出)

>  php test.php log

```
2017-12-09 14:03:44 PID:19690 [debug] master启动
```
