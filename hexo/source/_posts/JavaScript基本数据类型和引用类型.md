---
title: JavaScript基本数据类型和引用类型
tags: JavaScript
categories: JavaScript
comments: true
copyright: true
abbrlink: reference_type_and_basic_data_type_in_javascript
date: 2017-07-15 23:16:33
updated: 2017-07-15 23:16:33
---

JavaScript中具有5种基本的数据类型，这5种已经是不能再简化的类型：Number、String、Boolean、Null、Undefined。

引用类型也被称为复合类型，既可以包含基本数据类型，也可以包含其他复合类型。

**其中，基本类型将值直接存储在由标识符所分配的内存区域中，而复合类型仅存储指向包含实际对象数据的内存地址的引用。该引用也称为“指针”。** 所以，在JavaScript中将这两种类型的数据作为参数传递时，产生的实际效果是完全不同的。

<!--more-->

我们用一段测试代码来展现二者的不同：
```javascript
var num_1 = 10;
var num_2 = num_1;
num_2 = 11;

console.log(num_1);
console.log(num_2);


var obj_1 = {
	name: 'name_1',
	age: 12
};

var obj_2 = obj_1;
obj_2.name = 'name_2';

console.log(obj_1);
console.log(obj_2);
```

运行上述代码，我们会得到如下 的结果：
```c
10
11
{ name: 'name_2', age: 12 }
{ name: 'name_2', age: 12 }
```

可以看到，num_2 和num_1 这两个变量完全是分开和隔离的，而 obj_1 和 obj_2 则不是，我们对 obj_2 的内容进行修改，会导致 obj_1的改变，可以看出，**复制一个引用类型的变量，仅仅是复制了该变量中所保存的指向对象引用，而并未克隆其所指向的对象**。