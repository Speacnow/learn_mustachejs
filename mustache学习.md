

mustache.js github地址：https://github.com/janl/mustache.js

语法规则也基于这个包。

学习资料来源：http://www.atguigu.com/ ，https://www.bilibili.com/video/BV1iX4y1K72v

github代码：  https://github.com/Speacnow/learn_mustachejs

例：利用data数据将模板字符串解析为html字符串

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>

<body>
    <div id="container"></div>

    <script src="/xuni/bundle.js"></script>

    <script>
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
        `;

        // 数据
        var data = {
            students: [
                { 'name': '小明', 'hobbies': ['编程', '游泳'] },
                { 'name': '小红', 'hobbies': ['看书', '弹琴', '画画'] },
                { 'name': '小强', 'hobbies': ['锻炼'] }
            ]
        };

        // 调用render
        var domStr = SSG_TemplateEngine.render(templateStr, data);
        console.log(domStr);

        // 渲染上树
        var container = document.getElementById('container');
        container.innerHTML = domStr;
    </script>
</body>

</html>
```

#步骤流程

1. 创建scanner类，scanner初始指向模板字符串的开始

2. 将模板字符串变为tokens数组
3. 形成嵌套tokens
4. 解析嵌套tokens,利用递归形成html dom字符串

```js
// 0. 全局提供SSG_TemplateEngine对象
window.SSG_TemplateEngine = {
    // 渲染方法
    render(templateStr, data) {
        // 调用parseTemplateToTokens函数，让模板字符串能够变为tokens数组
        var tokens = parseTemplateToTokens(templateStr);
        //console.log(tokens);
        // 调用renderTemplate函数，让tokens数组变为dom字符串
        var domStr = renderTemplate(tokens, data);
        
        return domStr;
    }
};
/* 
	创建功能函数
    功能是可以在dataObj对象中，寻找用连续点符号的keyName属性
    比如，dataObj是
    {
        a: {
            b: {
                c: 100
            }
        }
    }
    那么lookup(dataObj, 'a.b.c')结果就是100
    不忽悠大家，这个函数是某个大厂的面试题
*/
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



/*
	1.创建scanner类，scanner初始指向模板字符串的开始，形成初始tokens
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

    // 功能弱，就是走过指定内容，没有返回值，就是跳过{{和}}两个符号
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



/* 
    2.将模板字符串变为tokens数组
*/
function parseTemplateToTokens(templateStr) {
    var tokens = [];
    // 创建扫描器
    var scanner = new Scanner(templateStr);
    var words;
    // 让扫描器工作
    while (!scanner.eos()) {
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
        // 过双大括号
        scanner.scan('{{');
        // 收集开始标记出现之前的文字
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
    /*
       此时的初始tokens是这样：
       [
        ["text", "<div><ul>"]
        ["#", "students"]
        ["text", "<li class="myli">学生"]
        ["name", "name"]
        ["text", "的爱好是<ol>"]
        ["#", "hobbies"]
        ["text", "<li>"]
        ["name", "."]
        ["text", "</li>"]
        ["/", "hobbies"]
        ["text", "</ol></li>"]
        ["/", "students"]
        ["text", "</ul></div>"]
       ]
       
    */
       return nestTokens(tokens);
}




/*
	3.形成嵌套tokens，注意栈的使用
*/
function nestTokens(tokens) {
    
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
    //console.log(nestedTokens);
    
    /*此时的tokens已经嵌套
    [
            ["text", "<div><ul>"]
            ["#", "students",
                [
                    ["text", "<li class="myli">学生"]
                    ["name", "name"]
                    ["text", "的爱好是<ol>"]
                    ["#", "hobbies",
                        [
                            ["text", "<li>"]
                            ["name", "."]
                            ["text", "</li>"]
                        ]
                    ]
                    
                    ["text", "</ol></li>"]
                ]    
            ]
            ["text", "</ul></div>"]
        ]  
    */

    return nestedTokens;
};



/*
	4.解析嵌套tokens,利用递归形成html dom字符串
	函数的功能是让tokens数组变为dom字符串
*/
function renderTemplate(tokens, data) {
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
function parseArray(token, data) {
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
            ...v[i],//将v[i]展开
            '.': v[i] // 添加.属性，满足一下情况
            /*
            				{{#hobbies}}
                            <li>{{.}}</li>
                            {{/hobbies}}
            */
        });
    }
    return resultStr;
};

```

