---
title: kotlin使用gson序列化遇到的坑
date: 2017-03-07 15:42:52
tags: [gson,kotlin]
categories: kotlin
---



先来介绍一下`kotlin`，`kotlin`是`JetBrains`开发的一个基于 JVM 的新的编程语言。如果你是一个正经的java开发者，那你应该不陌生，因为`IDEA`和`Android Studio`都是他家的产品。

至于为什么学习kotlin，kotlin有什么好处，以后再说，毕竟我现在也才接触而已。

<!--more-->

做Gson序列化，实体类是必不可少的，先写两个实体类压压惊。

```java
public class Res<T>{
        private String time;
        private String error;
        private String action;
        private String status;
        private T data;

    public String getTime() {
        return time;
    }

    public void setTime(String time) {
        this.time = time;
    }

    public String getError() {
        return error;
    }

    public void setError(String error) {
        this.error = error;
    }

    public String getAction() {
        return action;
    }

    public void setAction(String action) {
        this.action = action;
    }

    public String getStatus() {
        return status;
    }

    public void setStatus(String status) {
        this.status = status;
    }

    public T getData() {
        return data;
    }

    public void setData(T data) {
        this.data = data;
    }
}
```

```java
public class Yd{
    private Float weight;
    private Float muscle;
    private Float fat;
    private Float water;

    public Float getWeight() {
        return weight;
    }

    public void setWeight(Float weight) {
        this.weight = weight;
    }

    public Float getMuscle() {
        return muscle;
    }

    public void setMuscle(Float muscle) {
        this.muscle = muscle;
    }

    public Float getFat() {
        return fat;
    }

    public void setFat(Float fat) {
        this.fat = fat;
    }

    public Float getWater() {
        return water;
    }

    public void setWater(Float water) {
        this.water = water;
    }
}
```

这两个实体类加起来近百行代码，如果用kotlin来实现的话只有区区两行。

```kotlin
data class Res<out T>(val time:String, val error:String, val action:String, val status:String, val data:T)
data class Yd(val weight: Float,val muscle:Float,val fat:Float,val water:Float)
```

当然这都是题外话，和主题无关。

```java
public static void main(String[] arg){
    String json = "{\"action\":0,\"status\":0,\"time\":0,\"data\":[{\"weight\":0,\"muscle\":0,\"fat\":0,\"water\":0},{\"weight\":0,\"muscle\":0,\"fat\":0,\"water\":0},{\"weight\":0,\"muscle\":0,\"fat\":0,\"water\":0}]}";
    Type type = new TypeToken<Res<List<Yd>>>(){}.getType();
    Res<List<Yd>> res = new Gson().fromJson(json,type);
    System.out.println(res.getData().get(0).getFat());
}
```

这是一段熟悉的java代码，作用是将json数据解析成对象，这行代码的输出结果是0.0。

我把这段java code 改成kotlin code后却报了错。

`java.lang.ClassCastException: com.google.gson.internal.LinkedTreeMap cannot be cast to Yd`

```kotlin
fun main(arg: Array<String>) {
    val json = "{\"action\":0,\"status\":0,\"time\":0,\"data\":[{\"weight\":0,\"muscle\":0,\"fat\":0,\"water\":0},{\"weight\":0,\"muscle\":0,\"fat\":0,\"water\":0},{\"weight\":0,\"muscle\":0,\"fat\":0,\"water\":0}]}"
    val type = object : TypeToken<Res<List<Yd>>>() {}.type
    val res = Gson().fromJson<Res<List<Yd>>>(json, type)
    println(res.data[0].fat)
}
```

真的是求爷爷告奶奶，查遍了百度谷歌都找不到，后来在群里的大神的帮助下发现原来是实体类写泛型时多谢了一个`out`，当真是欲哭无泪啊。当时我写的时候是没有加out的，但是IDE却提示我加上，本着对他的信任，领着我走上了伤心路。

总结：不要盲目的相信IDE的提醒！！！



## 有问题反馈

在使用中有任何问题，欢迎反馈给我，可以用以下联系方式跟我交流

- Email: 2563892038@qq.com
- Github: [LiXiaoGuo](https://github.com/LiXiaoGuo)