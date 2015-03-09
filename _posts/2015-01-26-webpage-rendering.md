---
layout: post
category: blog
title: 每个前端工程师都应该知道的网页渲染知识
description:
---

由 [Alexander Skutin]() 于 2014年5月26日撰写

今天我将着重介绍网页渲染的知识和为什么在web开发领域它是如此的重要。涉及到这方面的文章已经非常多了，但信息比较分散和碎片化。要是想搞清楚这方面的知识，仅仅是举个例子，我不得不去从很多来源中去调研。这些正是我决定写这篇文章的原因。我相信这篇文章不但对初学者有用，对那些希望掌握更多知识以及巩固已有知识的高级开发者也同样有帮助。

因为样式和脚本在页面渲染过程中扮演者至关重要的角色，因此当页面布局被定义的时候，渲染应该在尽可能早的时候就开始优化。专业的人士有必要去了解一些特定的技巧去避免性能问题。

这篇文章并不会去详细的研究浏览器内部的机理，而是提供一些常用的准则。不同的浏览器引擎工作时也会不一样，这会使得特定于某个了浏览器的研究变得更加困难。

浏览器是如何渲染一个网页的
-----------------------
我们由一个浏览器在渲染一个网页时所做的一系列操作的概览开始：

1. 由从服务器接收到的HTML文本建立DOM（文档对象模型）树结构。
2. 样式被载入和解析，生成CSSOM（样式对象模型）
3. 在DOM和CSSOM的最基础上，一个渲染树被创建，它包含了一组即将被渲染的
   对象（webkit将它们称之为“renderer”或者“render object”，在Gecko中则被称为“frame”）。
   渲染树仅仅是除去那些不可见的元素（比如`<head>`标签和拥有`display:none`属性的元素；等等）的DOM树的一个反射。
   每个字段在渲染树中被当作一个单独的renderer。每个正在被渲染的对象都包含了它相应的附加了计算好的样式的DOM对象
  （或者是一个文本段落）。换句话说，渲染树描述了一个DOM树的视觉表现。
4. 渲染树上的每一个元素的坐标都被计算好，这被称之为“布局”。浏览器使用一种称为流的手段，这样仅仅需要一种途径就可以
   完成所有元素的布局（表格需要更多的途径）。
5. 最后，以上的结果被显示在浏览器的窗口中，这个操作被称之为“绘制”。

当用户同页面交互时，或者页面被脚本修改时，因为页面底层的结构发生了变化，上述所述的一些操作会被重复触发。

重绘
---
当修改一个元素的样式但没有影响到其在页面上的位置的时候（譬如`background-color`，`border-color`，`visibility`），浏览器仅仅将新的样式应用后重新绘制该元素（这意味着一次“重绘”或“样式更新”的发生）。

重排
---
当一次改变影响到了文档内容或结构，或者一个元素的位置，一次重排（或者说重新布局）就会发生。这些改变通常是由下面的行为触发的：

+ DOM操作（添加元素，删除，更改或者改变元素的排序）；
+ 内容改变，包括文本在表单域中的改变；
+ CSS属性的计算或更改；
+ 添加或更改样式表；
+ 更改“class”属性；
+ 浏览器窗口操作（例如窗口大小改变，滚动）；
+ 伪类的激活

浏览器是如何优化渲染的
-------------------
浏览器在尽其所有去将重绘/重排的区域限制在仅仅被更改的元素之上。例如，一个拥有absolute/fixed定位元素的大小改变只会印象到这个元素自身和它的后代，而将一个相似的改变应用到静止定位的元素上就会触发所有后面元素的重排。

另一个优化的技术就是当一段Javascript代码在运行时，浏览器缓存了它的更改，当代码运行结束时，再将所有的更改一次性的应用。例如，这段代码只会触发一次重排和重绘：


    var $body = $('body');  
    $body.css('padding', '1px'); // 重排，重绘  
    $body.css('color', 'red'); // 重绘 
    $body.css('margin', '2px'); // 重排，重绘
    // 事实上，只有一次重排，重绘发生。


然而，如同上面提到的一样，访问一个元素的属性会触发一次强制的重排。如果我们在先前的代码中额外的添加了一行去读取元素属性时就会发生。

    var $body = $('body');
    $body.css('padding', '1px');
    $body.css('padding'); // 读取一个属性，一次强制的重排
    $body.css('color', 'red');
    $body.css('margin', '2px');

结果，我们触发了2次重排而不是一次。因为这个原因，你应该将元素属性的读取工作组合在一起来优化性能（[看看一个JSBIN上更详细的例子](http://jsbin.com/duhah/2/edit)）。

有时候，你需要去触发一次强制的重排。例如：我们需要在同一个元素上应用一个相同的属性（例如`margin-left`）两次。首先，它应该在没有动画效果的情况下被设为`100px`，然后使用`transition`动画更改为`50px`。你应该[在JSBIN上](http://jsbin.com/qutev/1/edit)研究下这个例子，但我会在这儿仔细解释一下。

我们由创建一个拥有一个transition的CSS类开始：

    .has-transition {
       -webkit-transition: margin-left 1s ease-out;
          -moz-transition: margin-left 1s ease-out;
            -o-transition: margin-left 1s ease-out;
               transition: margin-left 1s ease-out;
    }

然后实现它：

    // 我们默认含有“has-transution”类的元素
    var $targetElem = $('#targetElemId');

    // 移除transition类名 
    $targetElem.removeClass('has-transition');

    // 显然类名不再存在，我们更改属性
    $targetElem.css('margin-left', 100);

    // 恢复transition类名
    $targetElem.addClass('has-transition');

    // 更改属性
    $targetElem.css('margin-left', 50);

然而这次实现并没有像预期那样工作。更改被缓存了并只在代码段落结束时才被应用。我们需要一次强制重绘，我们可以通过下面的更改来实现这一点：

    // 移除transition类名
    $(this).removeClass('has-transition');

    // 更改属性
    $(this).css('margin-left', 100);

    // 触发一次强制的重排，这样类名和属性的更改才会立即生效
    $(this)[0].offsetHeight; // 一个例子，其他的属性应该也会工做。

    // 恢复transition类名
    $(this).addClass('has-transition');

    // 更改属性
    $(this).css('margin-left', 50);

现在它按照预期工作了。

一些优化方面的实用建议
-------------------
综合可用的信息，我可以推荐下面这些手段：

+ 构建合法的HTML和CSS，不要忘记指定文档编码。样式表应该在`<head>`中被包含，脚本应该被插在`<body>`标签的最后。
+ 试着简化和优化CSS选择器（这经常普遍的被使用CSS预编译器的开发者忽略）。保持内嵌深度为最低。下面是CSS选择器的性能的排序（从最快开始）：
  1. ID选择器： `#id`
  2. 类选择器: `.class`
  3. 标签：`div`
  4. 相邻的兄弟选择器：`a+i`
  5. 父选择器：ul>li
  6. 全局选择器：`*`
  7. 属性选择器：`input[type="text"]`
  8. 伪类和伪元素：`a:hover` 你应当记住，浏览器是从右往左处理选择器的，这正是为什么右侧字符最多的选择器应当是最快的原因 - 或者`#id`或者`.class`：

      div * {...} // 糟糕   
      .list li {...} // 糟糕   
      .list-item {...} // 糟糕         
      #list .list-item {...} // 太棒了



1. 在脚本中 ，无论什么时候，尽可能的最小化DOM操作。缓存任何东西，包括属性和对象（如果它们会被复用的话）。当执行复杂的操作时，在“offline”元素上会更好（一个“offline”元素就是一个仅存储在内存中并且同DOM断开的元素），然后稍后将其插入DOM。
2. 如果你使用jQuery来选取元素，遵循[jQuery选择器最佳实践](http://learn.jquery.com/performance/optimize-selectors/)。
3. 要更改一个元素的样式，修改类名是效率最高的方法。执行此更改的元素在DOM树中越深，就会显得越好（同样源于这解耦了表现的逻辑）。
4. 如果可以，尽可能的在拥有absolute/fixed定位的元素上执行动画。
5. 在滚动时禁用复杂的`:hover`动画是一个好主意（例如，在`<body>`上添加一个额外的“no-hover”类）。[阅读一篇和这个主题相关的文章](http://habrahabr.ru/post/204238/)。

想了解更多，看看这些文章：

1. [浏览器是如何工作的]()
2. [渲染：重绘，重排/重新布局，重新应用样式](http://www.phpied.com/rendering-repaint-reflowrelayout-restyle/)

希望你会觉得这篇文章有用！


要转载这篇文章,请保留下面META信息,谢谢!  
[译文：http://kezhen.info/webpage-rendering](http://kezhen.info/webpage-rendering)  
[二道贩子：http://frontendbabel.info/articles/webpage-rendering-101/](http://frontendbabel.info/articles/webpage-rendering-101/)  
[原文：Рендеринг WEB-страницы: что об этом должен знать front-end разработчик](http://habrahabr.ru/post/224187/)
