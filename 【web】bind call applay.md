# bind(),call()和applay()的用法

1. 每个函数都包含两个非继承而来的方法：call()方法和apply()方法。

2. 相同点：这两个方法的作用是一样的。
   3.改变this的指向
   4.继承别的函数中的实例(对象冒充)
   <script>
   window.color = 'green';
   document.color = 'yellow';

   

   ```jsx
    var s1 = {color: 'blue' };
    function changeColor(){
        console.log(this.color);
    }
   
    changeColor.call();         //green (默认传递参数)
    changeColor.call(window);   //green
    changeColor.call(document); //yellow
    changeColor.call(this);     //green
    changeColor.call(s1);       //blue
   ```

   </script>

   //例2
   var Pet = {
   words : '...',
   speak : function (say) {
   console.log(say + ''+ this.words)
   }
   }
   Pet.speak('Speak'); // 结果：Speak...

   var Dog = {
   words:'Wang'
   }

   //将this的指向改变成了Dog
   Pet.speak.call(Dog, 'Speak'); //结果： SpeakWang

apply()方法使用示例：





```jsx
//例1
<script>
    window.number = 'one';
    html.number = 'two';
    function changeColor(){
        console.log(this.number);
    }

    changeColor.apply();         //one (默认传参)
    changeColor.apply(window);   //one
    changeColor.apply(html); //two
    changeColor.apply(this);     //one
    changeColor.apply(qq);       //three

</script>

//例2
function Pet(words){
    this.words = words;
    this.speak = function () {
        console.log( this.words)
    }
}
function Dog(words){
    //Pet.call(this, words); //结果： Wang
   Pet.apply(this, arguments); //结果： Wang
}
var dog = new Dog('Wang');
dog.speak();
```

call与apply的区别
1.第一个参数相同,后面的call要逐一列举apply放在数组中

例子1




```csharp
function add(c,d){
	return this.a + this.b + c + d;
}
var s = {a:1, b:2};
console.log(add.call(s,3,4)); // 1+2+3+4 = 10
console.log(add.apply(s,[5,6])); // 1+2+5+6 = 14
```

1. bind()方法会创建一个新函数，称为绑定函数，当调用这个绑定函数时，绑定函数会以创建它时传入 bind()方法的第一个参数作为 this，传入 bind() 方法的第二个以及以后的参数加上绑定函数运行时本身的参数按照顺序作为原函数的参数来调用原函数。

   ```
   var bar = function(){
   console.log(this.x);
   }
   var foo = {
   x:3
   }
   bar(); // undefined
   var func = bar.bind(foo);
   func(); // 3
   ```

   当你希望改变上下文环境之后并非立即执行，而是回调执行的时候，使用 bind() 方法。而 apply/call 则会立即执行函数。

最后总结:
**apply 、 call 、bind 三者都是用来改变函数的this对象的指向的；**
**apply 、 call 、bind 三者第一个参数都是this要指向的对象，也就是想指定的上下文；**
**apply 、 call 、bind 三者都可以利用后续参数传参；**
**bind 是返回对应函数，便于稍后调用；apply 、call 则是立即调用 。**



