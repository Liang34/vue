## `vue`源码探秘之`mustache`模板引擎

#### 什么是模板引擎

模板引擎是将数据变为视图最优雅的解决方案,比如我们在`vue`中常用的：

```html
<li v-for="item in arr"></li>// v-for就是模板引擎
<div>{{name}}</div> // {{}}也是模板引擎
```

`mustache`是最早的模板引擎库，其`{{}}`语法也被`Vue`沿用。

 `mustache`官方git： https://github.com/janl/mustache.js 

####  `mustache`的基本使用

- 循环：

```js
// 模板字符串
var templateStr = `
    <div>
        <ul>
            {{#students}}
            <li class="myli">
                学生{{name}}的爱好是
                <ol>
                    {{#hobbies}}
                    <li>{{.}}</li>
                    {{/hobbies}}
                </ol>
            </li>
            {{/students}}
        </ul>
    </div>
`
// 数据
var data = {
    students: [
        { 'name': '小明', 'hobbies': ['编程', '游泳'] },
        { 'name': '小红', 'hobbies': ['看书', '弹琴', '画画'] },
        { 'name': '小强', 'hobbies': ['锻炼'] }
    ]
}
// 调用render
var domStr = SSG_TemplateEngine.render(templateStr, data);
console.log(domStr)
// 渲染上树
var container = document.getElementById('container');
container.innerHTML = domStr;
```

实际效果：

![must](E:\primaryCode\vueCodeExe\part1_vue源码探秘之mustache模板引擎\imgs\must.png)

- 不循环：

![must2](E:\primaryCode\vueCodeExe\part1_vue源码探秘之mustache模板引擎\imgs\must2.png)

- 简单数组：

![must3](E:\primaryCode\vueCodeExe\part1_vue源码探秘之mustache模板引擎\imgs\must3.png)

- 布尔值：

```js
var templateStr = `{{#m}}<h1>你好</h1>{{/m}}`;
var data = {m: false};
var domStr = Mustache.render(templateStr, data);
// result
<h1>标签不会被渲染
```

#### `mustache`库的底层原理：

![must4](E:\primaryCode\vueCodeExe\part1_vue源码探秘之mustache模板引擎\imgs\must4.png)

那么`tokens`是什么？其实`tokens`就是一个js的嵌套数组，说白了就是模板字符串的JS表示。

![must5](E:\primaryCode\vueCodeExe\part1_vue源码探秘之mustache模板引擎\imgs\must5.png)

![must6](E:\primaryCode\vueCodeExe\part1_vue源码探秘之mustache模板引擎\imgs\must6.png)

如上面所示tokens是将模板字符串文本内容和变量（大括号里面的）划分开，并且起不同的名字`name`与`text`

 `mustache`库底层重点要做两个事情： 

① 将模板字符串编译为tokens形式 

② 将tokens结合数据，解析为`dom`字符串  

#### 第一步：实现模板字符串编译为tokens

- ##### 先不考虑循环嵌套的情况：

目标：

```js
var templateStr = `我买了一个{{thing}}，好{{mood}}啊`
// 在调用render方法后得到

```

分析：由上面的图我们可以知道，生成的tokens实际上是把每一个双大括号前面的划分开来，然后这部分作为一个数组，下标为0取做`text`，下标为1取为所截取到的文本内容，然后把双大括号的内容取出来最为一个新数组，下标0取做`name`下标1放变量名。这里主要用到`Scanner`类,里面的`scan`方法用于跳过双大括号，`ScannerUtil`用于扫描文字直到双大括号。

```js
/* 
    扫描器类
*/
class Scanner {
    constructor(templateStr) {
        // 将模板字符串写到实例身上
        this.templateStr = templateStr;
        // 指针
        this.pos = 0;
        // 尾巴，一开始就是模板字符串原文
        this.tail = templateStr;
    }

    // 功能弱，就是走过指定内容，没有返回值
    scan(tag) {
        if (this.tail.indexOf(tag) == 0) {
            // tag有多长，比如{{长度是2，就让指针后移多少位
            this.pos += tag.length;
            // 尾巴也要变，改变尾巴为从当前指针这个字符开始，到最后的全部字符
            this.tail = this.templateStr.substring(this.pos);
        }
    }

    // 让指针进行扫描，直到遇见指定内容结束，并且能够返回结束之前路过的文字
    scanUtil(stopTag) {
        // 记录一下执行本方法的时候pos的值
        const pos_backup = this.pos;
        // 当尾巴的开头不是stopTag的时候，就说明还没有扫描到stopTag
        // 写&&很有必要，因为防止找不到，那么寻找到最后也要停止下来
        while (!this.eos() && this.tail.indexOf(stopTag) != 0) {
            this.pos++;
            // 改变尾巴为从当前指针这个字符开始，到最后的全部字符
            this.tail = this.templateStr.substring(this.pos);
        }

        return this.templateStr.substring(pos_backup, this.pos);
    }

    // 指针是否已经到头，返回布尔值。end of string
    eos() {
        return this.pos >= this.templateStr.length;
    }
};
```

生成一个`tokens`

```js
function parseTemplateToTokens(templateStr) {
    const tokens = []
    const scanner = new Scanner(templateStr)
    let words
    while(!scanner.eos()){
        // 扫描text
        words = scanner.scanUtil('{{')
        // 存text
        tokens.push(['text', words])
        // 跳过大括号
        scanner.scan('{{')
        words = scanner.scanUtil('}}')
        scanner.scan('}}')
        tokens.push(['name', words])
    }
    return tokens
}
```

由此我们就完成了没有循环情况的第一步：

```
0: (2) ["text", "我买了一个"]
1: (2) ["name", "thing"]
2: (2) ["text", "，好"]
3: (2) ["name", "mood"]
4: (2) ["text", "啊"]
5: (2) ["name", ""]
```

- ##### 考虑嵌套的情况：

我们先改变一下对于`{{#student}}`以及`{{/student}}`的收集方法；

```js
function parseTemplateToTokens(templateStr) {
    const tokens = []
    const scanner = new Scanner(templateStr)
    let words
    while(!scanner.eos()){
        // 收集开始标记出现之前的文字
        words = scanner.scanUtil('{{');
        if (words != '') {
            // 尝试写一下去掉空格，智能判断是普通文字的空格，还是标签中的空格
            // 标签中的空格不能去掉，比如<div class="box">不能去掉class前面的空格
            let isInJJH = false;
            // 空白字符串
            var _words = '';
            for (let i = 0; i < words.length; i++) {
                // 判断是否在标签里
                if (words[i] == '<') {
                    isInJJH = true;
                } else if (words[i] == '>') {
                    isInJJH = false;
                }
                // 如果这项不是空格，拼接上
                if (!/\s/.test(words[i])) {
                    _words += words[i];
                } else {
                    // 如果这项是空格，只有当它在标签内的时候，才拼接上
                    if (isInJJH) {
                        _words += ' ';
                    }
                }
            }
            // 存起来，去掉空格
            tokens.push(['text', _words]);
        }
        // 跳过空格
        scanner.scan('{{')
        words = scanner.scanUtil('}}');
        if (words != '') {
            // 这个words就是{{}}中间的东西。判断一下首字符
            if (words[0] == '#') {
                // 存起来，从下标为1的项开始存，因为下标为0的项是#
                tokens.push(['#', words.substring(1)]);
            } else if (words[0] == '/') {
                // 存起来，从下标为1的项开始存，因为下标为0的项是/
                tokens.push(['/', words.substring(1)]);
            } else {
                // 存起来
                tokens.push(['name', words]);
            }
        }
        // 过双大括号
        scanner.scan('}}');
    }
```

此时的结果：

```js
0: (2) ["text", "<div><ul>"]
1: (2) ["#", "students"]
2: (2) ["text", "<li class=\"myli\">学生"]
3: (2) ["name", "name"]
4: (2) ["text", "的爱好是<ol>"]
5: (2) ["#", "hobbies"]
6: (2) ["text", "<li>"]
7: (2) ["name", "."]
8: (2) ["text", "</li>"]
9: (2) ["/", "hobbies"]
10: (2) ["text", "</ol></li>"]
11: (2) ["/", "students"]
12: (2) ["text", "</ul></div>"]
```

但是：我们需要的结果：

```js
0: (2) ["text", "<div><ul>"]
1: Array(3)
0: "#"
1: "students"
2: Array(5)
0: (2) ["text", "<li class=\"myli\">学生"]
1: (2) ["name", "name"]
2: (2) ["text", "的爱好是<ol>"]
3: (3) ["#", "hobbies", Array(3)]
4: (2) ["text", "</ol></li>"]
length: 5
__proto__: Array(0)
length: 3
__proto__: Array(0)
2: (2) ["text", "</ul></div>"]
```

如上面图所示，如果要考虑循环的情况将会出现多维数组嵌套的情况，这时我们可以借助栈来思考我们处于哪一层，

这里只要处理`#`与`/`

我们可以保存一个栈：遇到`#`就进栈，遇到`/`就出栈:参考实现：

```js
/* 
    函数的功能是折叠tokens，将#和/之间的tokens能够整合起来，作为它的下标为2的项
*/
function nestTokens(tokens){
    // 结果数组
    var nestedTokens = [];
    // 栈结构，存放小tokens，栈顶（靠近端口的，最新进入的）的tokens数组中当前操作的这个tokens小数组
    var sections = [];
    // 收集器，天生指向nestedTokens结果数组，引用类型值，所以指向的是同一个数组
    // 收集器的指向会变化，当遇见#的时候，收集器会指向这个token的下标为2的新数组
    var collector = nestedTokens;

    for (let i = 0; i < tokens.length; i++) {
        let token = tokens[i];

        switch (token[0]) {
            case '#':
                // 收集器中放入这个token
                collector.push(token);
                // 入栈
                sections.push(token);
                // 收集器要换人。给token添加下标为2的项，并且让收集器指向它
                collector = token[2] = [];
                break;
            case '/':
                // 出栈。pop()会返回刚刚弹出的项
                sections.pop();
                // 改变收集器为栈结构队尾（队尾是栈顶）那项的下标为2的数组
                collector = sections.length > 0 ? sections[sections.length - 1][2] : nestedTokens;
                break;
            default:
                // 甭管当前的collector是谁，可能是结果nestedTokens，也可能是某个token的下标为2的数组，甭管是谁，推入collctor即可。
                collector.push(token);
        }
    }

    return nestedTokens;
}
```

那么我们的第一步就完成了；

#### 第二步：将`tokens`与`data`相结合

其实我们只需要遍历`tokens`，tokens中的每一项token，当token[0]为`text`时直接拼接上结果文本即可，当`token[0]`为`name`时就把data[token[1]]替换上去即可。

首先我们要面对的一个问题是：如果我们的模板中出现这种情况

```vue
<div>{{a.b.c}}</div>
{
  a: {
	b: {
		c: 100
	}
  }
}
```

怎么将data中的数据换到`name`上去，显然是不能用`data[token[1]]`来进行替换的，因为会变成`data['a.b.c']`，显然，这个值是undefined。

这里我们写个lookup方法来处理：

```js
function lookup(dataObj, keyName) {
    // 看看keyName中有没有点符号，但是不能是.本身
    if (keyName.indexOf('.') != -1 && keyName != '.') {
        // 如果有点符号，那么拆开
        var keys = keyName.split('.');
        // 设置一个临时变量，这个临时变量用于周转，一层一层找下去。
        var temp = dataObj;
        // 每找一层，就把它设置为新的临时变量
        for (let i = 0; i < keys.length; i++) {
            temp = temp[keys[i]];
        }
        return temp;
    }
    // 如果这里面没有点符号
    return dataObj[keyName];
};
```

然后我们要面对的第二个问题是：

怎么去处理`#`号里面的内容呢？

```js
/* 
    处理数组，结合renderTemplate实现递归
    注意，这个函数收的参数是token！而不是tokens！
    token是什么，就是一个简单的['#', 'students', [

    ]]
    
    这个函数要递归调用renderTemplate函数，调用多少次？？？
    千万别蒙圈！调用的次数由data决定
    比如data的形式是这样的：
    {
        students: [
            { 'name': '小明', 'hobbies': ['游泳', '健身'] },
            { 'name': '小红', 'hobbies': ['足球', '蓝球', '羽毛球'] },
            { 'name': '小强', 'hobbies': ['吃饭', '睡觉'] },
        ]
    };
    那么parseArray()函数就要递归调用renderTemplate函数3次，因为数组长度是3
*/

export default function parseArray(token, data) {
    // 得到整体数据data中这个数组要使用的部分
    var v = lookup(data, token[1]);
    // 结果字符串
    var resultStr = '';
    // 遍历v数组，v一定是数组
    // 注意，下面这个循环可能是整个包中最难思考的一个循环
    // 它是遍历数据，而不是遍历tokens。数组中的数据有几条，就要遍历几条。
    for(let i = 0 ; i < v.length; i++) {
        // 这里要补一个“.”属性
        // 拼接
        resultStr += renderTemplate(token[2], {
            ...v[i],
            '.': v[i]
        });
    }
    return resultStr;
};
```

最后，总的渲染函数：

```js
/* 
    函数的功能是让tokens数组变为dom字符串
*/
export default function renderTemplate(tokens, data) {
    // 结果字符串
    var resultStr = '';
    // 遍历tokens
    for (let i = 0; i < tokens.length; i++) {
        let token = tokens[i];
        // 看类型
        if (token[0] == 'text') {
            // 拼起来
            resultStr += token[1];
        } else if (token[0] == 'name') {
            // 如果是name类型，那么就直接使用它的值，当然要用lookup
            // 因为防止这里是“a.b.c”有逗号的形式
            resultStr += lookup(data, token[1]);
        } else if (token[0] == '#') {
            resultStr += parseArray(token, data);
        }
    }

    return resultStr;
}
```

完结~





