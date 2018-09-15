---
layout: post
title:  "IntelliJ IDEA配置升级指南"
date:  2018-09-15 20:39:04
type: java
categories: [工具]
keywords: idea,material theme ui
---

## IntelliJ IDEA推荐设置

具体设置包括以下几个方面（详细步骤请参考原文[IntelliJ IDEA 推荐设置讲解](http://wiki.jikexueyuan.com/project/intellij-idea-tutorial/settings-recommend-introduce.html)）：
- 1.Auto Import配置，让导入包更智能
- 2.支持ctrl+鼠标放大屏幕，代码review的时候比较很方便
- 3.自定义快捷键
- 4.显示行号
- 5.Java代码注释风格修改
- 6.隐藏.idea目录

## IntelliJ主题优化
一直以来也是用的IDEA的默认浅色主题，默认提供的主题其实已经非常不错了。

最近，才突发奇想想换一个新鲜点的主题， [color-themes（IDEA主题）](http://color-themes.com/?view=index)网站上面找了很久，也没有找到满意的。
### IDEA主题导入
-   [color-themes（IDEA主题）](http://color-themes.com/?view=index) 下载主题jar
- 导入主题
![Alt text](../images/20180915192241.png)
由于不太适应dark系列的主题，且很多主题进去在细节上面处理总是感觉有点欠缺，最终也没能找到满意的主题。

直到发现了`Material Theme UI`，才发现原来IDEA的主题还有这么多选择。有种发现新大陆的感觉。只需要下载插件，各种主题配色，真的是很惊艳。
![Alt text](../images/20180915192824.png)

当然我依然喜欢浅色系，选择了`ONE LIGHT`主题配色，但是不排除以后会切换到其他的主题，支持一键切换，而且深色主题做的也真的是非常完美。
颜值如下图：
>个人习惯的浅色系：
![Alt text](../images/20180915233819.png)
深色系依然惊艳，重要的是支持快捷键切换
![Alt text](../images/20180915234110.png)


## IntelliJ推荐插件
- 1.Alibaba Java Coding Guidelines(阿里巴巴出的代码规范检查插件)
>强烈推荐，真的是太惊艳了，最近发现最好的插件，做代码规范首推！

![Alt text](../images/20180915194349.png)

- 2.Key promoter(快捷键提示统计，快捷键是IDEA的亮点之一)
- 3.GsonFormat	把 JSON 字符串直接实例化成类
- 4.Material Theme UI:这个也是强烈推荐，支持好多优美的主题，装上后写代码的欲望爆棚。
- 5.String Manipulation:驼峰式命名和下划线命名交替变化
- 6.Rainbow Brackets:对各个对称括号进行着色，方便查看
- 7.Grep Console:自定义设置控制台输出颜色
可以将日志按级别输出不同的颜色，方便查阅

![Alt text](../images/20180915225353.png)
- 8.IdeaVim 习惯vim可以装这个插件
- 9.Bytecode Viewer 查看字节码的插件，对于学习jvm很有帮助
![Alt text](../images/20180915233207.png)

## IntelliJ IDEA 其他黑科技
- 1.live Template
- 2.添加背景(不过这个貌似没有太大的意义，好玩而已）

## IntelliJ 快捷键设置
- 1.第一步首先要禁用掉搜狗输入法的快捷键，搜狗输入法快捷键没有太大的必要，且很多和idea的快捷键冲突。
- 2.定制自己的快捷键，这个看个人喜好
>
<h2 id="2b61e0d5977f2e38f06e16281c802b47">Ctrl</h2>
<table>
<thead>
<tr>
<th style="text-align: left;">快捷键</th>
<th style="text-align: left;">介绍</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>F</kbd></td>
<td style="text-align: left;">在当前文件进行文本查找 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>R</kdb></td>
<td style="text-align: left;">在当前文件进行文本替换 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Z</kdb></td>
<td style="text-align: left;">撤销 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Y</kdb></td>
<td style="text-align: left;">删除光标所在行 或 删除选中的行 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>X</kdb></td>
<td style="text-align: left;">剪切光标所在行 或 剪切选择内容</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>C</kdb></td>
<td style="text-align: left;">复制光标所在行 或 复制选择内容</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>D</kdb></td>
<td style="text-align: left;">复制光标所在行 或 复制选择内容，并把复制内容插入光标位置下面 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>W</kdb></td>
<td style="text-align: left;">递进式选择代码块。可选中光标所在的单词或段落，连续按会在原有选中的基础上再扩展选中范围 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>E</kdb></td>
<td style="text-align: left;">显示最近打开的文件记录列表 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>N</kdb></td>
<td style="text-align: left;">根据输入的 <strong>类名</strong> 查找类文件 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>G</kdb></td>
<td style="text-align: left;">在当前文件跳转到指定行处</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>J</kdb></td>
<td style="text-align: left;">插入自定义动态代码模板 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>P</kdb></td>
<td style="text-align: left;">方法参数提示显示 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Q</kdb></td>
<td style="text-align: left;">光标所在的变量 / 类名 / 方法名等上面（也可以在提示补充的时候按），显示文档内容</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>U</kdb></td>
<td style="text-align: left;">前往当前光标所在的方法的父类的方法 / 接口定义 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>B</kdb></td>
<td style="text-align: left;">进入光标所在的方法/变量的接口或是定义处，等效于 <code>Ctrl + 左键单击</code>  <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>K</kdb></td>
<td style="text-align: left;">版本控制提交项目，需要此项目有加入到版本控制才可用</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>T</kdb></td>
<td style="text-align: left;">版本控制更新项目，需要此项目有加入到版本控制才可用</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>H</kdb></td>
<td style="text-align: left;">显示当前类的层次结构</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>O</kdb></td>
<td style="text-align: left;">选择可重写的方法</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>I</kdb></td>
<td style="text-align: left;">选择可继承的方法</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>+</kdb></td>
<td style="text-align: left;">展开代码</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>-</kdb></td>
<td style="text-align: left;">折叠代码</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>/</kdb></td>
<td style="text-align: left;">注释光标所在行代码，会根据当前不同文件类型使用不同的注释符号 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>[</kdb></td>
<td style="text-align: left;">移动光标到当前所在代码的花括号开始位置</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>]</kdb></td>
<td style="text-align: left;">移动光标到当前所在代码的花括号结束位置</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>F1</kdb></td>
<td style="text-align: left;">在光标所在的错误代码处显示错误信息 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>F3</kdb></td>
<td style="text-align: left;">调转到所选中的词的下一个引用位置 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>F4</kdb></td>
<td style="text-align: left;">关闭当前编辑文件</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>F8</kdb></td>
<td style="text-align: left;">在 Debug 模式下，设置光标当前行为断点，如果当前已经是断点则去掉断点</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>F9</kdb></td>
<td style="text-align: left;">执行 Make Project 操作</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>F11</kdb></td>
<td style="text-align: left;">选中文件 / 文件夹，使用助记符设定 / 取消书签 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>F12</kdb></td>
<td style="text-align: left;">弹出当前文件结构层，可以在弹出的层上直接输入，进行筛选</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Tab</kdb></td>
<td style="text-align: left;">编辑窗口切换，如果在切换的过程又加按上delete，则是关闭对应选中的窗口</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>End</kdb></td>
<td style="text-align: left;">跳到文件尾</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Home</kdb></td>
<td style="text-align: left;">跳到文件头</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Space</kdb></td>
<td style="text-align: left;">基础代码补全，默认在 Windows 系统上被输入法占用，需要进行修改，建议修改为 <code>Ctrl + 逗号</code> <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Delete</kdb></td>
<td style="text-align: left;">删除光标后面的单词或是中文句 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>BackSpace</kdb></td>
<td style="text-align: left;">删除光标前面的单词或是中文句 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>1,2,3...9</kdb></td>
<td style="text-align: left;">定位到对应数值的书签位置 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>左键单击</kdb></td>
<td style="text-align: left;">在打开的文件标题上，弹出该文件路径 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>光标定位</kdb></td>
<td style="text-align: left;">按 Ctrl 不要松开，会显示光标所在的类信息摘要</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>左方向键</kdb></td>
<td style="text-align: left;">光标跳转到当前单词 / 中文句的左侧开头位置 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>右方向键</kdb></td>
<td style="text-align: left;">光标跳转到当前单词 / 中文句的右侧开头位置 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>前方向键</kdb></td>
<td style="text-align: left;">等效于鼠标滚轮向前效果 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>后方向键</kdb></td>
<td style="text-align: left;">等效于鼠标滚轮向后效果 <code>（必备）</code></td>
</tr>
</tbody>
</table>
<h2 id="a2e92861b757ab878312dd57993d60cf">Alt</h2>
<table>
<thead>
<tr>
<th style="text-align: left;">快捷键</th>
<th style="text-align: left;">介绍</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align: left;"><kbd>Alt</kbd> + <kbd>`</kbd>|显示版本控制常用操作菜单弹出层 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Alt</kbd> + <kbd>Q</kbd></td>
<td style="text-align: left;">弹出一个提示，显示当前类的声明 / 上下文信息</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Alt</kbd> + <kbd>F1</kbd></td>
<td style="text-align: left;">显示当前文件选择目标弹出层，弹出层中有很多目标可以进行选择 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Alt</kbd> + <kbd>F2</kbd></td>
<td style="text-align: left;">对于前面页面，显示各类浏览器打开目标选择弹出层</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Alt</kbd> + <kbd>F3</kbd></td>
<td style="text-align: left;">选中文本，逐个往下查找相同文本，并高亮显示</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Alt</kbd> + <kbd>F7</kbd></td>
<td style="text-align: left;">查找光标所在的方法 / 变量 / 类被调用的地方</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Alt</kbd> + <kbd>F8</kbd></td>
<td style="text-align: left;">在 Debug 的状态下，选中对象，弹出可输入计算表达式调试框，查看该输入内容的调试结果</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Alt</kbd> + <kbd>Home</kbd></td>
<td style="text-align: left;">定位 / 显示到当前文件的 <code>Navigation Bar</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Alt</kbd> + <kbd>Enter</kbd></td>
<td style="text-align: left;">IntelliJ IDEA 根据光标所在问题，提供快速修复选择，光标放在的位置不同提示的结果也不同 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Alt</kbd> + <kbd>Insert</kbd></td>
<td style="text-align: left;">代码自动生成，如生成对象的 set / get 方法，构造函数，toString() 等 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Alt</kbd> + <kbd>左方向键</kbd></td>
<td style="text-align: left;">切换当前已打开的窗口中的子视图，比如Debug窗口中有Output、Debugger等子视图，用此快捷键就可以在子视图中切换 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Alt</kbd> + <kbd>右方向键</kbd></td>
<td style="text-align: left;">按切换当前已打开的窗口中的子视图，比如Debug窗口中有Output、Debugger等子视图，用此快捷键就可以在子视图中切换 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Alt</kbd> + <kbd>前方向键</kbd></td>
<td style="text-align: left;">当前光标跳转到当前文件的前一个方法名位置 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Alt</kbd> + <kbd>后方向键</kbd></td>
<td style="text-align: left;">当前光标跳转到当前文件的后一个方法名位置 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Alt</kbd> + <kbd>1,2,3...9</kbd></td>
<td style="text-align: left;">显示对应数值的选项卡，其中 1 是 Project 用得最多 <code>（必备）</code></td>
</tr>
</tbody>
</table>
<h2 id="825a3d98017bab11815ad2817201324c">Shift</h2>
<table>
<thead>
<tr>
<th style="text-align: left;">快捷键</th>
<th style="text-align: left;">介绍</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align: left;"><kbd>Shift</kbd> + <kbd>F1</kbd></td>
<td style="text-align: left;">如果有外部文档可以连接外部文档</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Shift</kbd> + <kbd>F2</kbd></td>
<td style="text-align: left;">跳转到上一个高亮错误 或 警告位置</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Shift</kbd> + <kbd>F3</kbd></td>
<td style="text-align: left;">在查找模式下，查找匹配上一个</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Shift</kbd> + <kbd>F4</kbd></td>
<td style="text-align: left;">对当前打开的文件，使用新Windows窗口打开，旧窗口保留</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Shift</kbd> + <kbd>F6</kbd></td>
<td style="text-align: left;">对文件 / 文件夹 重命名</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Shift</kbd> + <kbd>F7</kbd></td>
<td style="text-align: left;">在 Debug 模式下，智能步入。断点所在行上有多个方法调用，会弹出进入哪个方法</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Shift</kbd> + <kbd>F8</kbd></td>
<td style="text-align: left;">在 Debug 模式下，跳出，表现出来的效果跟 <code>F9</code> 一样</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Shift</kbd> + <kbd>F9</kbd></td>
<td style="text-align: left;">等效于点击工具栏的 <code>Debug</code> 按钮</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Shift</kbd> + <kbd>F10</kbd></td>
<td style="text-align: left;">等效于点击工具栏的 <code>Run</code> 按钮</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Shift</kbd> + <kbd>F11</kbd></td>
<td style="text-align: left;">弹出书签显示层 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Shift</kbd> + <kbd>Tab</kbd></td>
<td style="text-align: left;">取消缩进 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Shift</kbd> + <kbd>ESC</kbd></td>
<td style="text-align: left;">隐藏当前 或 最后一个激活的工具窗口</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Shift</kbd> + <kbd>End</kbd></td>
<td style="text-align: left;">选中光标到当前行尾位置</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Shift</kbd> + <kbd>Home</kbd></td>
<td style="text-align: left;">选中光标到当前行头位置</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Shift</kbd> + <kbd>Enter</kbd></td>
<td style="text-align: left;">开始新一行。光标所在行下空出一行，光标定位到新行位置 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Shift</kbd> + <kbd>左键单击</kbd></td>
<td style="text-align: left;">在打开的文件名上按此快捷键，可以关闭当前打开文件 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Shift</kbd> + <kbd>滚轮前后滚动</kbd></td>
<td style="text-align: left;">当前文件的横向滚动轴滚动 <code>（必备）</code></td>
</tr>
</tbody>
</table>
<h2 id="e3025837a0cb66dbc536ae73f4f55fda">Ctrl + Alt</h2>
<table>
<thead>
<tr>
<th style="text-align: left;">快捷键</th>
<th style="text-align: left;">介绍</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Alt</kbd> + <kbd>L</kbd></td>
<td style="text-align: left;">格式化代码，可以对当前文件和整个包目录使用 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Alt</kbd> + <kbd>O</kbd></td>
<td style="text-align: left;">优化导入的类，可以对当前文件和整个包目录使用 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Alt</kbd> + <kbd>I</kbd></td>
<td style="text-align: left;">光标所在行 或 选中部分进行自动代码缩进，有点类似格式化</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Alt</kbd> + <kbd>T</kbd></td>
<td style="text-align: left;">对选中的代码弹出环绕选项弹出层 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Alt</kbd> + <kbd>J</kbd></td>
<td style="text-align: left;">弹出模板选择窗口，将选定的代码加入动态模板中</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Alt</kbd> + <kbd>H</kbd></td>
<td style="text-align: left;">调用层次</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Alt</kbd> + <kbd>B</kbd></td>
<td style="text-align: left;">在某个调用的方法名上使用会跳到具体的实现处，可以跳过接口</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Alt</kbd> + <kbd>C</kbd></td>
<td style="text-align: left;">重构-快速提取常量</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Alt</kbd> + <kbd>F</kbd></td>
<td style="text-align: left;">重构-快速提取成员变量</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Alt</kbd> + <kbd>V</kbd></td>
<td style="text-align: left;">重构-快速提取变量</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Alt</kbd> + <kbd>Y</kbd></td>
<td style="text-align: left;">同步、刷新</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Alt</kbd> + <kbd>S</kbd></td>
<td style="text-align: left;">打开 IntelliJ IDEA 系统设置 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Alt</kbd> + <kbd>F7</kbd></td>
<td style="text-align: left;">显示使用的地方。寻找被该类或是变量被调用的地方，用弹出框的方式找出来</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Alt</kbd> + <kbd>F11</kbd></td>
<td style="text-align: left;">切换全屏模式</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Alt</kbd> + <kbd>Enter</kbd></td>
<td style="text-align: left;">光标所在行上空出一行，光标定位到新行 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Alt</kbd> + <kbd>Home</kbd></td>
<td style="text-align: left;">弹出跟当前文件有关联的文件弹出层</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Alt</kbd> + <kbd>Space</kbd></td>
<td style="text-align: left;">类名自动完成</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Alt</kbd> + <kbd>左方向键</kbd></td>
<td style="text-align: left;">退回到上一个操作的地方 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Alt</kbd> + <kbd>右方向键</kbd></td>
<td style="text-align: left;">前进到上一个操作的地方 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Alt</kbd> + <kbd>前方向键</kbd></td>
<td style="text-align: left;">在查找模式下，跳到上个查找的文件</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Alt</kbd> + <kbd>后方向键</kbd></td>
<td style="text-align: left;">在查找模式下，跳到下个查找的文件</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Alt</kbd> + <kbd>右括号（]）</kbd></td>
<td style="text-align: left;">在打开多个项目的情况下，切换下一个项目窗口</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Alt</kbd> + <kbd>左括号（[）</kbd></td>
<td style="text-align: left;">在打开多个项目的情况下，切换上一个项目窗口</td>
</tr>
</tbody>
</table>
<h2 id="0bf82ba6d05ffabff2073c42657647e5">Ctrl + Shift</h2>
<table>
<thead>
<tr>
<th style="text-align: left;">快捷键</th>
<th style="text-align: left;">介绍</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>F</kbd></td>
<td style="text-align: left;">根据输入内容查找整个项目 或 指定目录内文件 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>R</kbd></td>
<td style="text-align: left;">根据输入内容替换对应内容，范围为整个项目 或 指定目录内文件 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>J</kbd></td>
<td style="text-align: left;">自动将下一行合并到当前行末尾 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>Z</kbd></td>
<td style="text-align: left;">取消撤销 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>W</kbd></td>
<td style="text-align: left;">递进式取消选择代码块。可选中光标所在的单词或段落，连续按会在原有选中的基础上再扩展取消选中范围 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>N</kbd></td>
<td style="text-align: left;">通过文件名定位 / 打开文件 / 目录，打开目录需要在输入的内容后面多加一个正斜杠 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>U</kbd></td>
<td style="text-align: left;">对选中的代码进行大 / 小写轮流转换 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>T</kbd></td>
<td style="text-align: left;">对当前类生成单元测试类，如果已经存在的单元测试类则可以进行选择 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>C</kbd></td>
<td style="text-align: left;">复制当前文件磁盘路径到剪贴板 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>V</kbd></td>
<td style="text-align: left;">弹出缓存的最近拷贝的内容管理器弹出层</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>E</kbd></td>
<td style="text-align: left;">显示最近修改的文件列表的弹出层</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>H</kbd></td>
<td style="text-align: left;">显示方法层次结构</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>B</kbd></td>
<td style="text-align: left;">跳转到类型声明处 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>I</kbd></td>
<td style="text-align: left;">快速查看光标所在的方法 或 类的定义</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>A</kbd></td>
<td style="text-align: left;">查找动作 / 设置</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>/</kbd></td>
<td style="text-align: left;">代码块注释 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>[</kbd></td>
<td style="text-align: left;">选中从光标所在位置到它的顶部中括号位置 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>]</kbd></td>
<td style="text-align: left;">选中从光标所在位置到它的底部中括号位置 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>+</kbd></td>
<td style="text-align: left;">展开所有代码 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>-</kbd></td>
<td style="text-align: left;">折叠所有代码 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>F7</kbd></td>
<td style="text-align: left;">高亮显示所有该选中文本，按Esc高亮消失 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>F8</kbd></td>
<td style="text-align: left;">在 Debug 模式下，指定断点进入条件</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>F9</kbd></td>
<td style="text-align: left;">编译选中的文件 / 包 / Module</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>F12</kbd></td>
<td style="text-align: left;">编辑器最大化 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>Space</kbd></td>
<td style="text-align: left;">智能代码提示</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>Enter</kbd></td>
<td style="text-align: left;">自动结束代码，行末自动添加分号 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>Backspace</kbd></td>
<td style="text-align: left;">退回到上次修改的地方 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>1,2,3...9</kbd></td>
<td style="text-align: left;">快速添加指定数值的书签 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>左键单击</kbd></td>
<td style="text-align: left;">把光标放在某个类变量上，按此快捷键可以直接定位到该类中 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>左方向键</kbd></td>
<td style="text-align: left;">在代码文件上，光标跳转到当前单词 / 中文句的左侧开头位置，同时选中该单词 / 中文句 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>右方向键</kbd></td>
<td style="text-align: left;">在代码文件上，光标跳转到当前单词 / 中文句的右侧开头位置，同时选中该单词 / 中文句 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>前方向键</kbd></td>
<td style="text-align: left;">光标放在方法名上，将方法移动到上一个方法前面，调整方法排序 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>后方向键</kbd></td>
<td style="text-align: left;">光标放在方法名上，将方法移动到下一个方法前面，调整方法排序 <code>（必备）</code></td>
</tr>
</tbody>
</table>
<h2 id="79a7ade344e6321bb49e9fe2e928d46d">Alt + Shift</h2>
<table>
<thead>
<tr>
<th style="text-align: left;">快捷键</th>
<th style="text-align: left;">介绍</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align: left;"><kbd>Alt</kbd> + <kbd>Shift</kbd> + <kbd>N</kbd></td>
<td style="text-align: left;">选择 / 添加 task <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Alt</kbd> + <kbd>Shift</kbd> + <kbd>F</kbd></td>
<td style="text-align: left;">显示添加到收藏夹弹出层 / 添加到收藏夹</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Alt</kbd> + <kbd>Shift</kbd> + <kbd>C</kbd></td>
<td style="text-align: left;">查看最近操作项目的变化情况列表</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Alt</kbd> + <kbd>Shift</kbd> + <kbd>I</kbd></td>
<td style="text-align: left;">查看项目当前文件</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Alt</kbd> + <kbd>Shift</kbd> + <kbd>F7</kbd></td>
<td style="text-align: left;">在 Debug 模式下，下一步，进入当前方法体内，如果方法体还有方法，则会进入该内嵌的方法中，依此循环进入</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Alt</kbd> + <kbd>Shift</kbd> + <kbd>F9</kbd></td>
<td style="text-align: left;">弹出 <code>Debug</code>  的可选择菜单</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Alt</kbd> + <kbd>Shift</kbd> + <kbd>F10</kbd></td>
<td style="text-align: left;">弹出 <code>Run</code>  的可选择菜单</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Alt</kbd> + <kbd>Shift</kbd> + <kbd>左键双击</kbd></td>
<td style="text-align: left;">选择被双击的单词 / 中文句，按住不放，可以同时选择其他单词 / 中文句 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Alt</kbd> + <kbd>Shift</kbd> + <kbd>前方向键</kbd></td>
<td style="text-align: left;">移动光标所在行向上移动 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Alt</kbd> + <kbd>Shift</kbd> + <kbd>后方向键</kbd></td>
<td style="text-align: left;">移动光标所在行向下移动 <code>（必备）</code></td>
</tr>
</tbody>
</table>
<h2 id="7589cafe7a42413c095670c012d39e80">Ctrl + Shift + Alt</h2>
<table>
<thead>
<tr>
<th style="text-align: left;">快捷键</th>
<th style="text-align: left;">介绍</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>Alt</kbd> + <kbd>V</kbd></td>
<td style="text-align: left;">无格式黏贴 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>Alt</kbd> + <kbd>N</kbd></td>
<td style="text-align: left;">前往指定的变量 / 方法</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>Alt</kbd> + <kbd>S</kbd></td>
<td style="text-align: left;">打开当前项目设置 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>Alt</kbd> + <kbd>C</kbd></td>
<td style="text-align: left;">复制参考信息</td>
</tr>
</tbody>
</table>
<h2 id="0d98c74797e49d00bcc4c17c9d557a2b">其他</h2>
<table>
<thead>
<tr>
<th style="text-align: left;">快捷键</th>
<th style="text-align: left;">介绍</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align: left;"><kbd>F2</kbd></td>
<td style="text-align: left;">跳转到下一个高亮错误 或 警告位置 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>F3</kbd></td>
<td style="text-align: left;">在查找模式下，定位到下一个匹配处</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>F4</kbd></td>
<td style="text-align: left;">编辑源 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>F7</kbd></td>
<td style="text-align: left;">在 Debug 模式下，进入下一步，如果当前行断点是一个方法，则进入当前方法体内，如果该方法体还有方法，则不会进入该内嵌的方法中</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>F8</kbd></td>
<td style="text-align: left;">在 Debug 模式下，进入下一步，如果当前行断点是一个方法，则不进入当前方法体内</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>F9</kbd></td>
<td style="text-align: left;">在 Debug 模式下，恢复程序运行，但是如果该断点下面代码还有断点则停在下一个断点上</td>
</tr>
<tr>
<td style="text-align: left;"><kbd>F11</kbd></td>
<td style="text-align: left;">添加书签 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>F12</kbd></td>
<td style="text-align: left;">回到前一个工具窗口 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>Tab</kbd></td>
<td style="text-align: left;">缩进 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>ESC</kbd></td>
<td style="text-align: left;">从工具窗口进入代码文件窗口 <code>（必备）</code></td>
</tr>
<tr>
<td style="text-align: left;"><kbd>连按两次Shift</kbd></td>
<td style="text-align: left;">弹出 <code>Search Everywhere</code> 弹出层</td>
</tr>
</tbody>
</table>


## 参考文档

[极客学院IntelliJ IDEA教程](http://wiki.jikexueyuan.com/project/intellij-idea-tutorial/)

[github.com/ChrisRM/material-theme-jetbrains#configuration](https://github.com/ChrisRM/material-theme-jetbrains#configuration)

[material-theme-ui](https://plugins.jetbrains.com/plugin/8006-material-theme-ui)





