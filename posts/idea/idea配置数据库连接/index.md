# Idea配置数据库连接


idea配置数据库连接时，提示如下错误

![20210127-idea-servertimezone](/images/20210127-idea-servertimezone.png)  

```shell
Server returns invalid timezone. Need to set 'serverTimezone' property.
```

点击 set timezone

默认的 serverTimezone 为 UTC

![20210127-默认serverTimezone](/images/20210127-默认serverTimezone.png)  

utc 为协调世界时，又称世界统一时间、世界标准时间，不属于任何时区，这套时间系统被应用于许多互联网和万维网的标准中，例如，网络时间协议就是协调世界时在互联网中使用的一种方式。

中国大陆、中国香港、中国澳门、中国台湾、蒙古国、新加坡、马来西亚、菲律宾、西澳大利亚州的时间与UTC的时差均为+8，也就是UTC+8。

但是，配置成 UTC+8 会报错：

```shell
[S1009] No timezone mapping entry for 'UTC+8'
com.mysql.cj.exceptions.WrongArgumentException: No timezone mapping entry for 'UTC+8'
```

查看mysql文档中对时区的支持：https://dev.mysql.com/doc/refman/8.0/en/time-zone-support.html

查看到mysql支持的timezone是根据操作系统判断的

```shell
[root@vultrguest ~]# ls /usr/share/zoneinfo/
Africa      Australia  Cuba     Etc      GMT-0      Indian       Kwajalein    MST7MDT  Portugal    ROC        Universal     zone.tab
America     Brazil     EET      Europe   GMT0       Iran         leapseconds  Navajo   posix       ROK        US            Zulu
Antarctica  Canada     Egypt    GB       Greenwich  iso3166.tab  Libya        NZ       posixrules  Singapore  UTC
Arctic      CET        Eire     GB-Eire  Hongkong   Israel       MET          NZ-CHAT  PRC         Turkey     WET
Asia        Chile      EST      GMT      HST        Jamaica      Mexico       Pacific  PST8PDT     tzdata.zi  W-SU
Atlantic    CST6CDT    EST5EDT  GMT+0    Iceland    Japan        MST          Poland   right       UCT        zone1970.tab
[root@vultrguest ~]# ls /usr/share/zoneinfo/Asia/
Aden       Bahrain     Chongqing  Gaza         Jerusalem    Kuala_Lumpur  Novokuznetsk  Rangoon        Tashkent       Ulan_Bator
Almaty     Baku        Chungking  Harbin       Kabul        Kuching       Novosibirsk   Riyadh         Tbilisi        Urumqi
Amman      Bangkok     Colombo    Hebron       Kamchatka    Kuwait        Omsk          Saigon         Tehran         Ust-Nera
Anadyr     Barnaul     Dacca      Ho_Chi_Minh  Karachi      Macao         Oral          Sakhalin       Tel_Aviv       Vientiane
Aqtau      Beirut      Damascus   Hong_Kong    Kashgar      Macau         Phnom_Penh    Samarkand      Thimbu         Vladivostok
Aqtobe     Bishkek     Dhaka      Hovd         Kathmandu    Magadan       Pontianak     Seoul          Thimphu        Yakutsk
Ashgabat   Brunei      Dili       Irkutsk      Katmandu     Makassar      Pyongyang     Shanghai       Tokyo          Yangon
Ashkhabad  Calcutta    Dubai      Istanbul     Khandyga     Manila        Qatar         Singapore      Tomsk          Yekaterinburg
Atyrau     Chita       Dushanbe   Jakarta      Kolkata      Muscat        Qostanay      Srednekolymsk  Ujung_Pandang  Yerevan
Baghdad    Choibalsan  Famagusta  Jayapura     Krasnoyarsk  Nicosia       Qyzylorda     Taipei         Ulaanbaatar
[root@vultrguest ~]# 
```

使用Aisa/Shanghai就可以了

![配置上海时区](/images/20210127-配置上海时区.png)  


扩展：

jdbc中的url里也有时区配置，示例如下

```
url: jdbc:mysql://127.0.0.1:3306/simple_demo?useSSL=false&useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai
```

此处serverTimezone也是指的mysql时区，也可以写为GMT+8，即

```
url: jdbc:mysql://127.0.0.1:3306/simple_demo?useSSL=false&useUnicode=true&characterEncoding=utf8&serverTimezone=GMT%2B8
```

GMT时间就是英国格林威治时间，也就是世界标准时间，是本初子午线上的地方时，是0时区的区时，与我国的标准时间北京时间（东八区）相差8小时，即晚8小时，故可以使用GMT+8表示北京时间。


问题来了，Aisa/Shanghai 和 GMT+8 的时间相同吗？

参考：https://www.zhihu.com/question/270709626/answer/357054228

```java
    public static void main(String[] args) throws ParseException {
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        sdf.setTimeZone(TimeZone.getTimeZone("Asia/Shanghai"));
        Date date = sdf.parse("1988-08-13 13:13:13");
        TimeZone.setDefault(TimeZone.getTimeZone("Asia/Shanghai"));
        System.out.println("date = " + date.toString()); //date = Sat Aug 13 13:13:13 CDT 1988
        System.out.println("date = " + date.getTime()); //date = 587448793000

        sdf.setTimeZone(TimeZone.getTimeZone("GMT+8"));
        date = sdf.parse("1988-08-13 13:13:13");
        TimeZone.setDefault(TimeZone.getTimeZone("GMT+8"));
        System.out.println("date = " + date.toString()); //date = Sat Aug 13 13:13:13 GMT+08:00 1988
        System.out.println("date = " + date.getTime()); //date = 587452393000

        /*
        时间戳相差一个小时
        587452393000 - 587448793000 = 3600000 毫秒 = 1 小时

        中国曾经实行过夏令时，使用Asia/Shanghai时会处理夏令时。使用GMT+8时不会处理夏令时，因为GMT+8不能确定是哪个国家
        1992年以后就没有这个问题了
         */
    }
```

所以，可以认为 Aisa/Shanghai 和 GMT+8 的时间是相同的。
