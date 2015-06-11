# 怎样定义js类

author:秋凡

datetime:2015-06-11 10:05:55

要解决的问题：
遇到要用js解析数据，并填充网页，如果过程式的写js，会很容易导致变量冲突，而且不利于迁移和维护，但是对于js的OOP编程几乎一窍不通，不过OOP的思想总归是和其他语言相通的，经过查找资料，和自己的摸索，此文主要记录摸索过程，通过实践验证理论。

### js类的定义

首页，我们不看广告，看疗效。
简单看看定义js类的方式

```javascript
function Util(){
    // ...
}

var help = function(){
    // ...
}

var obj = {};
```
<pre>
Util 既是函数，有可以是一个类，因为js里面函数和类的定义方式相同。
help 是另一种定义函数的方式。
obj 定义了一个对象。
</pre>


<script type="text/javascript">
function ObjAAA(){

    var _a1 = 1;
    this._a2 = 2;
    // console.log('construct');
    // console.log('inner a1 ' + _a1);
    // console.log('inner a2 ' + this._a2);

    this.fn3 = function(){
        return _a1;
    };

    var fn4 = function(){
        return 4;
    };

    function fn5(){
        return 5;
    };

}

var aaa = new ObjAAA();

console.log( 'aaa.a1 ' + aaa._a1 );
console.log( 'aaa.a2 ' + aaa._a2 );
console.log( 'aaa.fn3 ' + aaa.fn3() );
// console.log( 'ObjAAA.fn3 ' + ObjAAA.fn3() );
// console.log( 'aaa.fn4 ' + aaa.fn4() );
// console.log( 'aaa.fn5 ' + aaa.fn5() );
// console.log( 'ObjAAA.fn5 ' + ObjAAA.fn5() );
console.log('--------------');


var dddd = {
    'a':1,
    b:function(){
        return this.a;
    },
    cc:function(){
        return '111';
    },
    dd:function(){
        console.log('222');
    }
}

console.log('dddd.a ' + dddd.a);
console.log('dddd.b ' + dddd.b());
console.log('dddd.cc ' + dddd.cc());
dddd.dd();
console.log('--------------');


function Class1(){
    //self(self被附加到了对象上) self只对私有成员可见(能.点出来 i aa() .点不出来public_dd())
    var self = this;
    this.i = 1;
    this.aa = function(){
        this.i ++;
        console.log(this.i);
    }
    var private_bb = function(){
        console.log(self.i);
        //self.public_dd();//错误 self无法从外部访问,同时self也无法被这个对象的公共方法所访问
        //aa();//错误  私有方法要通过self调用
        public_dd();//可以直接调用 不能用self.public_dd();
        self.aa();
    }
    this.cc = function(){
        private_bb();//私有函数
    }

    //可以直接调用
    //  对象的公共方法
    function public_dd()
    {
        self.aa();
        console.log("dd");
    }
}

// var o = new Class1();//调用Class1构造函数不运行++(初始化没有调用不运行)
// o.cc();//运行++
// document.write(o.i);//return 2


</script>




