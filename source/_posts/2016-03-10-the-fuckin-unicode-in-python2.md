---
layout: post  
title:  "锟斤拷烫烫烫"  
category: news
author: "diggzhang"
---

Python中的字符编码问题是千年老妖。想搞懂锟斤拷的心，真是烫烫烫。

缘起是因在洗数据过程中，遇到这样一组字符：

```
[
    "å«å¹´çº§ä¸",
    "ä¸å¹´çº§ä¸",
    "八年级下",
    "ä¸å¹´çº§ä¸",
    "七年级下",
    "å«å¹´çº§ä¸",
    "七年级上",
    "八年级上"
]

```

没问题，大丈夫，遇到这样的乱码问题，按照一贯办法用`"str".decode('utf8')`和`"str".encode('utf8')`解决。原因是在Python2内部，字符串使用`unicode`编码，
所以只需要将乱码的字符decode成unicode后，再encode成utf8就应该可以解决。然而事实如下:

```

顺带一题，平时为了避开编码问题，平时都会在文件头加入如下三行，Python2跑起来后，设置系统默认的编码
import sys
reload(sys)
sys.setdefaultencoding('utf8')

>>> print "å«å¹´çº§ä¸".decode('utf8')
å«å¹´çº§ä¸
>>>
>>> print "å«å¹´çº§ä¸".decode('gbk')
氓聟芦氓鹿麓莽潞搂盲赂聥
>>>
>>> print "å«å¹´çº§ä¸".decode('gb2312')
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
UnicodeDecodeError: 'gb2312' codec can't decode bytes in position 2-3: illegal multibyte sequence

```

终于，这一天，这个python新手终于因为字符编码问题彻底懵逼了。

目前可以确认的是`UnicodeDecodeError`足以证明解码方式不对。必须确认这堆刚出道就打码的字符是什么，从数据库中distinct出这组list得出:

```

db= MongoClient(host='127.0.0.1', port=27017, connect=False)['yangcong']
feedbacks = db['feedbacks']
hypervideos = db['hypervideos']

semesterList = feedbacks.distinct("semester")
print(semsterList)

for semester in semsterList:
    print semester

//输出结果:
[
    u'\xe5\x85\xab\xe5\xb9\xb4\xe7\xba\xa7\xe4\xb8\x8b', u'\xe4\xb8\x83\xe5\xb9\xb4\xe7\xba\xa7\xe4\xb8\x8b', u'\u516b\u5e74\u7ea7\u4e0b', u'\xe4\xb8\x83\xe5\xb9\xb4\xe7\xba\xa7\xe4\xb8\x8a', u'\u4e03\u5e74\u7ea7\u4e0b', u'\xe5\x85\xab\xe5\xb9\xb4\xe7\xba\xa7\xe4\xb8\x8a', u'\u4e03\u5e74\u7ea7\u4e0a', u'\u516b\u5e74\u7ea7\u4e0a'
]
å «å¹´çº§ä¸ 
ä¸ å¹´çº§ä¸ 
八年级下
ä¸ å¹´çº§ä¸ 
七年级下
å «å¹´çº§ä¸ 
七年级上
八年级上

>>> print u'\u516b\u5e74\u7ea7\u4e0b'
八年级下
>>> print u'\xe5\x85\xab\xe5\xb9\xb4\xe7\xba\xa7\xe4\xb8\x8b'
å«å¹´çº§ä¸

```

两种编码混在一起了，对比可以看出`\xe`开头的字符一律乱码。这是个线索。我现在有一个线索和一颗混乱的心，然后就没有然后了。

……

…………

………………

解铃还须系铃人，最终前端好兄贵一段js代码处理了这堆乱码。大概处理思路是，先hex到16进制，再转出成utf8。
我们最终做了一个`[乱码：字符]`的映射表替换了数据库中的所有乱码。

映射表制作:

```

var semersList = [
    "å«å¹´çº§ä¸",
    "ä¸å¹´çº§ä¸",
    "八年级下",
    "ä¸å¹´çº§ä¸",
    "七年级下",
    "å«å¹´çº§ä¸",
    "七年级上",
    "八年级上"
]

mapObjArray = []

semersList.forEach(function (elem) {
    console.log('-----')
    console.log(elem)
    // console.log(EncodeUtf8(elem))
    console.log(decodeUtf8_in_Url(elem))
    mapObj = {}
    mapObj[elem] = decodeUtf8_in_Url(elem)

    console.log(mapObj)
    mapObjArray.push(mapObj)
    console.log('-----')

});

console.log(mapObjArray)

 function EncodeUtf8(s1)
  {
      var s = escape(s1);
      var sa = s.split("%");
      var retV ="";
      if(sa[0] != "")
      {
         retV = sa[0];
      }
      for(var i = 1; i < sa.length; i ++)
      {
           if(sa[i].substring(0,1) == "u")
           {
               retV += Hex2Utf8(Str2Hex(sa[i].substring(1,5)));

           }
           else retV += "%" + sa[i];
      }

      return retV;
  }
  function Str2Hex(s)
  {
      var c = "";
      var n;
      var ss = "0123456789ABCDEF";
      var digS = "";
      for(var i = 0; i < s.length; i ++)
      {
         c = s.charAt(i);
         n = ss.indexOf(c);
         digS += Dec2Dig(eval(n));

      }
      //return value;
      return digS;
  }
  function Dec2Dig(n1)
  {
      var s = "";
      var n2 = 0;
      for(var i = 0; i < 4; i++)
      {
         n2 = Math.pow(2,3 - i);
         if(n1 >= n2)
         {
            s += '1';
            n1 = n1 - n2;
          }
         else
          s += '0';

      }
      return s;

  }
  function Dig2Dec(s)
  {
      var retV = 0;
      if(s.length == 4)
      {
          for(var i = 0; i < 4; i ++)
          {
              retV += eval(s.charAt(i)) * Math.pow(2, 3 - i);
          }
          return retV;
      }
      return -1;
  }
  function Hex2Utf8(s)
  {
     var retS = "";
     var tempS = "";
     var ss = "";
     if(s.length == 16)
     {
         tempS = "1110" + s.substring(0, 4);
         tempS += "10" +  s.substring(4, 10);
         tempS += "10" + s.substring(10,16);
         var sss = "0123456789ABCDEF";
         for(var i = 0; i < 3; i ++)
         {
            retS += "%";
            ss = tempS.substring(i * 8, (eval(i)+1)*8);



            retS += sss.charAt(Dig2Dec(ss.substring(0,4)));
            retS += sss.charAt(Dig2Dec(ss.substring(4,8)));
         }
         return retS;
     }
     return "";
  }

  function Str2Hex(s)
  {
      var c = "";
      var n;
      var ss = "0123456789ABCDEF";
      var digS = "";
      for(var i = 0; i < s.length; i ++)
      {
         c = s.charAt(i);
         n = ss.indexOf(c);
         digS += Dec2Dig(eval(n));

      }
      //return value;
      return digS;
  }

  function decodeUtf8_in_Url(s1)
  {
      // escape函数用于对除英文字母外的字符进行编码。如“Visit W3School!”->"Visit%20W3School%21"
      var s = escape(s1);
      var sa = s.split("%");//sa[1]=u6211
      var retV ="";
      if(sa[0] != "")
      {
          retV = sa[0];
      }
      for(var i = 1; i < sa.length; i ++)
      {
          if(sa[i].substring(0,1) == "u")
          {
              retV += Hex2Utf8(Str2Hex(sa[i].substring(1,5)));
              if(sa[i].length>=6)
              {
                  retV += sa[i].substring(5);
              }
          }
          else retV += "%" + sa[i];
      }
      return decodeURI(retV);   // 强制告诉从URL中拿到的中文是utf-8编码，转码成URI后在解码URI，成为中文进行网络传输
  }

```
