*作者：UpSGE*

### 写在前面

本文以CMCC-EDU为例，实现自动认证。其他使用web认证的网络可以参考。

不想看原理解析，直接跳转到[抄作业](#应用（抄作业）)

### 抓包

连接网络，进入认证页面http://192.168.253.6/

打开dev-tools，监听网络连接，输入账号密码认证

查看第一条连接，得到以下信息

![alt text](https://pic.imgdb.cn/item/67495627d0e0a243d4dac073.png)

复制为curl，得到（已隐去隐私信息）

    curl ^"http://192.168.253.6:801/eportal/?c=Portal^&a=login^&callback=dr1003^&login_method=1^&user_account=^%^2C0^%^2CXXXX^%^40cmcc^&user_password=XXXX^&wlan_user_ip=XXXX^&wlan_user_ipv6=^&wlan_user_mac=XXXX^&wlan_ac_ip=192.168.253.5^&wlan_ac_name=^&jsVersion=3.3.2^&v=7455^" ^
      -H ^"Accept: */*^" ^
      -H ^"Accept-Language: zh-CN,zh;q=0.9,ja;q=0.8^" ^
      -H ^"Connection: keep-alive^" ^
      -H ^"Cookie: PHPSESSID=XXXX^" ^
      -H ^"Referer: http://192.168.253.6/^" ^
      -H ^"User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36 Edg/131.0.0.0^" ^
      --insecure

### 分析

提取可能的关键参数

    user_account
    user_password
    wlan_user_ip
    wlan_user_mac
    Cookie

经测试，关键参数及格式如下

    user_account=,0,学号@cmcc
    user_password=密码
    wlan_user_ip=ip地址

最简的curl命令如下

    curl "http://192.168.253.6:801/eportal/?wlan_ac_ip=192.168.253.5&c=Portal&a=login&callback=dr1003&login_method=1&user_account=,0,${学号}@cmcc&user_password=${密码}&wlan_user_ip=${ip地址}"

并且最终获得授权的设备ip为curl命令中wlan_user_ip参数的ip，与发包ip无关

### 应用（抄作业）

至此，便可以配合ping命令检测网络，ip命令获取ip，来实现自动认证

完整代码

    #!/bin/sh
    #检测网络
    check() {
        if ping -c 1 -W 5 8.8.8.8 > /dev/null 2>&1;
        then
            return 0
        else
            return 1
        fi
    }
    #认证
    auth() {
        #获取ip
        ip=$(ip addr show CMCC | grep "inet " | awk '{print $2}' | cut -d/ -f1)
        #学号
        user=""
        #密码
        pwd=""
        #记录日志
        echo $ip >> auth-cmcc.log
        #认证
        curl "http://192.168.253.6:801/eportal/?wlan_ac_ip=192.168.253.5&c=Portal&a=login&callback=dr1003&login_method=1&user_account=,0,${user}@cmcc&user_password=${pwd}&wlan_user_ip=${ip}" >> auth-cmcc.log
    }
    #判断是否能联网，若不能则进行认证
    if ! check; then
        auth
    fi

利用cron实现自动执行

    #7:00到22:59每分钟执行一次
    * 7-22 * * * /path/to/auth-cmcc.sh