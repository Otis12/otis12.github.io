---
title: PFORTIFIER
date: 2025-09-12 10:47:36
categories: [论文阅读]
tags: [Php, Security, 论文解读, gadget, 反序列化]
---

  下周小讨论班又又又轮到我讲论文了hh。刚好准备开始好好做一点技术积累之类的，反正都要做ppt了，那就顺便写个文章吧。

- **论文标题**:  PFORTIFIER: Mitigating PHP Object Injection through Automatic Patch Generation
- **作者**:  Bo Pang
- **发表时间**:  2025
- **期刊或会议名称**:  S&P
- **领域**:  Security



### 基础概念

  初次接触php中反序列化漏洞以及gadget利用，稍微花一些篇幅给自己讲讲，学学一些基础的概念。

**PHP 魔方方法**：魔方方法是PHP语言中特有的，无需手动调用满足条件自动触发的特殊方法（名字以`__`开头）。个人感觉同其他高级语言中的构造函数和析构函数很类似，只是有更广泛的定义和使用场景。

  举几个例子：

|   __destruct()   | 当对象被销毁（如脚本执行结束、`unset()`删除对象）时自动调用，核心作用是**清理对象占用的资源**（如关闭数据库连接、释放文件句柄）。 |
| :--------------: | ------------------------------------------------------------ |
|  **__wakeup()**  | **当对象被反序列化（`unserialize()`）时自动调用，核心作用是恢复对象的正常状态（如重新连接数据库、初始化缺失的属性）。** |
|   **__call()**   | **当调用对象中不存在或不可访问的方法时自动调用，核心作用是优雅处理方法调用错误（如返回默认值、记录错误日志）。** |
| **__toString()** | **当对象被当作字符串使用（如`echo $对象`、字符串拼接、JSON 编码）时自动调用，核心作用是定义对象转字符串的格式。** |



**PHP Object Injection (POI) 漏洞:**  POI 漏洞是攻击者通过篡改反序列化的对象，构造恶意方法调用链（gadget 链），执行非预期代码的漏洞。此类漏洞触发的前提是应用中存在用户可控的反序列化入口（比如用`unserialize()`处理用户输入的 Cookie、参数）。



**反序列化：**“ 序列化是将对象状态转换为可保持或可传输的格式的过程。与序列化相对的是反序列化，它将流转换为对象。这两个过程结合起来，可以轻松地存储和传输数据。”

  *原文：Serialization and deserialization are two fundamental mechanisms in PHP that allow developers to convert objects into a storable format, and vice versa. During serialization, an object is converted into a string that can be stored in a database or transmitted over the network. Deserialization, on the other hand, refers to the process of reconstructing objects from the serialized strings.*



**Gadget链：**在反序列化漏洞场景下，Gadget通常指向一类特定的代码片段，是攻击者可控对象能够触发的单个方法/函数，也是一个可被利用的最小单元。比如上文提到的PHP魔方方法很多都会被作为Gadget来实现攻击。多个Gadget按照顺序组合最终流向能够直接实现攻击效果的危险函数就形成了一个Gadget链。



**一个例子**：花了很久在背景知识了解上，倘若要先讲清楚一些Gadget本身开发者希望中的用途，文章会更加冗长。在这里我们不妨略过这部分，暂且认为每一个Gadget链中的部分都有正常的功能设计和用途。

```
class PendingBroadcast {
    protected $events; // 事件调度器（正常是Illuminate\Contracts\Events\Dispatcher实例）
    protected $event;  // 待广播的事件对象（如OrderPaidEvent）

    public function __construct($events, $event) {
        $this->events = $events; // 注入事件调度器
        $this->event = $event;   // 注入具体事件
    }

    // 魔法方法：对象销毁时自动触发（正常功能：确保广播任务执行）
    // 此处就是一个Gadget
    public function __destruct() {
        $this->events->dispatch($this->event); // 调用调度器的dispatch()分发事件
    }
}
```

**正常逻辑**：当`PendingBroadcast`对象被销毁（如请求结束）`__destruct()`会调用事件调度器的`dispatch()`，将`$event`分发给所有监听器（如发送短信、更新库存）




```
class Generator {
    protected $formatters = []; // 存储“方法名→回调函数”的映射（正常是格式化工具，如date_formatter）

    // 魔法方法：调用不存在的方法时触发（正常功能：动态处理未定义方法）
    // 此处也是一个Gadget
    public function __call($method, $attributes) { 
        return $this->format($method, $attributes); // 转发到format()
    }
	// 此处也是一个Gadget		
    public function format($formatter, $arguments = []) {
        // 正常功能：根据$formatter调用对应的格式化函数（如date($arguments)）
        return call_user_func_array($this->getFormatter($formatter), $arguments);//最后的sink点
    }

    protected function getFormatter($formatter) {
        return $this->formatters[$formatter]; // 从formatters中获取回调函数
    }
}
```

**正常逻辑**：当调用Generator不存在的方法（如$generator->date('Y-m-d')），__call()会根据$formatter（如date）从$formatters中获取对应函数并且尝试去执行。本身设计是省去开发者去定义测试数据的生成函数，改为动态调用依赖库中的预定义函数。


  总之，此漏洞还存在一个用户可控的反序列化入口，形如:

```
$maliciousObject = unserialize($_COOKIE['object']); 
```

  那么攻击者要做的是通过一些设计，巧妙地链接这些Gadget，使之成为一条完整的Gadget链条，最终在sink点执行我们想要的恶意命令。

```
// 步骤1：创建Generator对象，篡改formatters属性（关键！）
$poc_generator = new Generator();  // 新建一个Generator对象
// 给formatters赋值：把"dispatch"这个方法名，映射到"system"函数（system是PHP执行系统命令的函数）
$poc_generator->formatters = ["dispatch" => "system"];  // 这里利用了“protected属性可被赋值”的语法（攻击者通过序列化篡改）
// 步骤2：创建PendingBroadcast对象，篡改events属性（关键！）
$poc_broadcast = new PendingBroadcast();  // 新建PendingBroadcast对象
// 把原本该存“事件调度器”的events属性，改成上面的恶意Generator对象
$poc_broadcast->events = $poc_generator;  // 利用“属性可任意赋值”的语法，替换成恶意对象
// 把原本该存“事件”的event属性，改成攻击者想执行的命令（比如"whoami"，查看当前用户）
$poc_broadcast->event = "whoami";
```

​    *PHP 允许 “给对象的属性赋值为任意类型”—— 正常情况下`$poc_broadcast->events`应该是 “事件调度器” 对象，但攻击者改成了`Generator`对象；`$poc_broadcast->event`应该是 “事件” 对象，攻击者改成了字符串 “whoami”。*



  当 Laravel 反序列化攻击者注入的字符串后，会自动执行以下流程:

   1.反序列化：重建恶意对象 

2.  触发`__destruct()`: PHP 脚本执行结束后，会自动销毁内存里的对象（垃圾回收机制），当`$obj`（PendingBroadcast 对象）被销毁时：自动调用`$obj->__destruct()`方法,```__destruct()```里的代码```$this->events->dispatch($this->event)```开始执行：

   `$this`指当前的 PendingBroadcast 对象，`$this->events`就是之前篡改的`Generator`对象；

   `$this->events->dispatch(...)`：调用`Generator`对象的`dispatch()`方法，参数是 “whoami”。

3. 触发`__call()`: `Generator`类里没有`dispatch()`方法（正常情况下 Generator 不会有这个方法），所以触发`Generator`的`__call()`方法：

   `__call()`的参数`$method`就是 “dispatch”（调用的不存在的方法名）；

   `__call()`的参数`$attributes`就是`["whoami"]`（调用`dispatch()`时传的参数）；

   `__call()`里的代码`return $this->format($method, $attributes)`执行：调用`Generator`的`format()`方法，传参 “dispatch” 和 “whoami”。

4. 执行`format()`→调用`system()`（PHP “动态函数调用” 的语法）

   `Generator`的`format()`方法开始执行：

   - 第一步：调用`$this->getFormatter($format)`，`$format`是 “dispatch”；

   - `getFormatter()`里的代码`return $this->formatters[$format]`：从`formatters`数组里拿 “dispatch” 对应的 value，也就是 “system”（攻击者之前篡改的）；

   - 第二步：调用```call_user_func_array("system", ["whoami"])```

       第一个参数 “system”：要执行的函数（PHP 执行系统命令的函数）；

       第二个参数`["whoami"]`：传给`system`的参数；

   - 最终执行`system("whoami")`，返回当前服务器的用户（比如 “www-data”），攻击者成功执行命令。

   此漏洞的修复方式是在`Generator`类里加了`__wakeup()`方法，清空formatter数组来防止恶意映射。

   ```
   public function __wakeup(){
       $this->formatters = [];  // 清空formatters数组
   }
   ```

   ![image-20250914130957736](C:\Users\10749\AppData\Roaming\Typora\typora-user-images\image-20250914130957736.png)



### 文章研究背景

  1.PHP项目相当流行，市场占额达到了77.4%。

  2.PHP项目中很容易受到POI漏洞的影响，并且影响巨大，这是由于POI往往会导致RCE。

  3.基于以上两点，针对PHP项目中POI漏洞的修复就很重要了，现有的工作关于漏洞修补的方法由于Gadget链执行的不可预测性很难迁移。另一些现有工作则聚焦在检测Gadget链而没有关注补丁生成的部分。



### 文章面临的挑战

  在文章的研究背景下就可以推测出这篇工作主要的目标是针对于PHP项目中POI漏洞进行自动化的补丁生成。那么进行这样的补丁生成面临着一些技术上的挑战：

  **C1：**无限组合。Gadget链的不可预测执行可能涉及类方法的任意组合。这主要是由于存在用户可控的PHP对象属性以及PHP语言支持的动态方法调用导致的。因为可能的执行路径很难进行统计预测，因此很难在所有可能的执行路径上应用Sanitizer。

  *原文：C1: Infinite combinations. The unpredictable execution of gadget chains may involve arbitrary combinations of class methods. It is difficult to apply sanitizers on all possible execution paths.*

  **C2：**链式检测不完整且效率低下。目前的Gadget链检测方法存在覆盖率和分析效率不足的问题，制约了修补机制的全面性和有效性。提高链式覆盖率和效率也存在困难。

  *原文：C2: Incomplete and inefficient chain detection. Current gadget chain detection methods suffer from inadequate coverage and analysis efficiency, restricting the comprehensiveness and effectiveness of the patching mechanism. Improving chain coverage and efficiency is also difficult.*

  **C3：**影响最小化。文章的目标是修补 POI 漏洞，同时在修补后的应用中保持正常功能。

  *原文：C3: Impact minimization. We aim to patch POI vulnerabilities while maintaining normal functionalities in the patched applications, which is another challenge to be addressed.*

  并且由于PHP的开发中，所谓的魔方方法本身是PHP的一个重要特性。它既可能是开发者去实现某一功能所依赖的特性，也可能成为攻击者的攻击入口。因此不能将所有的PHP魔方方法一棍子打死，如果为每个可能存在被利用的魔方方法都生成Santizer，很有可能应用的正常功能就会受到影响。



### 本文核心贡献

  1.提出第一个POI漏洞修复方法。 

  2.提出了一种快速检测PHP项目中Gadget链的方法，较现有工作覆盖率提升了近40%，速度快10倍。

  3.证明了提出方法的有效性，Evaluation的部分。（不太爱看这部分，除非要参考实验，我一般直接略过）



### 本文核心发现

  **Possible Methods：** 攻击者可通过能控制的对象’如篡改后的`$this->logger`）触发的类方法本文定义为一个Possible Method。

  *原文: We further define class methods that may get triggered through an attacker controllable object as Possible Methods (PM) (i.e., getLog() and __call() in Line 9 and 15),*

**一个例子：**

```
<?php
class LogPrinter{
public function __destruct(){ // entry method
echo $this->logger->getLog(); // PM call site
	}
}

class Logger{
public function getLog(){ // expected method
return $this->log; 
	}
}

class SystemInfo{
public function __call(){ // unexpected method
$info = system($this->cmd); // sink method: cmd injection vul
return $info;
	}
}

$maliciousObject = unserialize($_COOKIE[’object’]); //POI Vul
// $maliciousObject = O:10:"SystemInfo":1:{s:3:"cmd";s:6:"whoami";
```

  **发现1：**大部分的攻击链是由一个entry、一些PM和多个Gadget所构成的。

  **发现2：**针对于POI漏洞的修复，往往在Gadget链执行阶段进行修复更加有效。

![image-20250914134308710](C:\Users\10749\AppData\Roaming\Typora\typora-user-images\image-20250914134308710.png)



  *As illustrated above, POI exploitation involves two key stages: payload injection and gadget chain execution (Figure 2). In the payload injection stage, the attacker gains control over a serialized string, allowing him/her to insert a meticulously crafted object through the deserialization method . In the gadget chain execution stage, the attackers invoke the entry method, steering the PHP applications toward security-sensitive sinks.*

  第一这是由于在第一个阶段，也就是Trigger Phase，针对攻击者输入的payload做过滤，在传统的web漏洞方向如Sqli、XSS等具有一定可行性。但针对于POI漏洞，由于反序列化的方法众多，很难在这个阶段去限制所有的反序列化操作并且去净化攻击者的payload。

  第二是由于Web 开发人员普遍使用反序列化来实现自定义功能，例如读取文件和恢复 cookie 等。由于二级开发者的安全意识有限，自定义代码更容易受到攻击，因此难以通过限制对象注入进行有效修补。

  第三是Gadget链在框架和库中的普遍存在。由于单个框架或库可能被多个二级开发人员采用，因此通过限制Gadget链来修补 POI 可以更有效地增强 PHP 应用程序的安全性。



### PFORTIFIER OverView

![image-20250914135123453](C:\Users\10749\AppData\Roaming\Typora\typora-user-images\image-20250914135123453.png)

  

  PPFORTIFIER 这个框架主要分成了三个模块：code summarization、simulated execution和patch generation。这里感觉没必要花很多篇幅讲了，主要的工作流比较简单。先通过静态分析进行一个代码总结，将PHP代码转为ast并且提取一些关键信息做成索引表，然后模拟执行找出潜在风险的Gadget链，最后通过策略生成补丁或者补丁建议。引用一下原文：

  ***Code Summarization**: In this step, PFORTIFIER parses the source code into abstract syntax trees (ASTs) and extracts essential information from them, including the fields, methods, class inheritances, interface inheritances, interface implementations, and trait inheritances.* 

  **Simulated Execution**： In this step, PFORTIFIER simulates the execution of ASTs, enabling the identification of potential gadget chains.* 

  ***Patch Generation:** In this step, PFORTIFIER attempts to patch the detected gadget chains by restricting the corresponding unsafe jumps caused by PM calls. It iteratively searches for the first patchable jump in the sequence of method calls and adds related checks to prevent unsafe jumps.*



#### **Code Summarization**

  这部分的核心目标是将PHP代码解析为结构化的表示，提取嘞、方法、继承关系等信息，为后续的的分析提供基础。

  **输入:**PHP源代码

  **输出：**两张映射表，一张是类名对应它所有方法的AST，一张是方法名对应所有实现了该方法的类。



#### Simulated Execution

  在 PFortifier 框架中， simulated execution 的本质是 “静态模拟 PHP 代码执行流程，通过追踪攻击者可控对象，最终检测出能触发危险行为的 gadget 链”。

  **输入:** “code summarization” 阶段生成的抽象语法树和attr_func_dict（方法名→实现类），无需重复解析源码，直接基于结构化信息模拟。

  **输出:** “检测到的 gadget 链列表”（含入口方法、PM 调用节点、sink 函数），后续 Patch Generation 阶段只需针对链中的 PM 节点（中间核心节点）添加限制，即可阻断攻击。

  那么作者们认为整个模拟执行并不是一个简单的工作，他们遇到了一些问题并且给出了一些解决方案和优化措施。

1. 当模拟执行遇到可能被赋值为污点对象的变量时，需要将污点对象的状态赋值给变量。现有工作的解决方法大多是建立一张指针流图通过传递指针关系来使得污点进行传播。但这篇工作认为针对大型的php项目建立指针流图的开销过大，几乎是不可接受的。采用php深拷贝来传递状态又会导致添加的污点状态丢失（php深拷贝语法不支持拷贝元数据）。那么他们提出了“**根对象相对索引**”的方法。

​	**根对象相对索引：**通过维护从反序列化入口对象到当前属性的完整访问路径，所有分支共享同一个根对象，确保污点标记在分支间正确传播。（也就是在给变量赋值进行模拟执行的时候，赋的就是这样一个路径+对象属性来避免拷贝，达成的效果就是不同的分支的可能被污染的变量指向的是同一个污点对象）

  2.分支合并问题，这个基于贪心策略，优先保留可能被污染的分支来做一个保守性分析，非常直观简单，不赘述。

  3.针对一个方法在项目中有多个实现，导致需要重复分析开销大提出了优化。通过静态预分析为PHP的多态方法和魔术方法（如call、wakeup）生成行为摘要，避免运行时的重复分析。

行为摘要包括：**输入输出关系**：比如调用 getLog (\(param)时，返回值是不是由param 决定的？

​			   **污点传播规则**：如果输入参数是攻击者可控的 “污点”，返回值会不会也变成污点？会不会传到其他属性里？

​			    **副作用清单**：比如__call () 会不会偷偷执行 system ()、file_put_contents () 这些危险函数？有没有修改全局变量？

  4.剩下是一些比较直观的优化，比如通过预定义好的安全检查函数来筛出假阳性，不多次执行循环来提高效率等。



#### Patch Generation

  这部分到了补丁生成的部分了。首先论文把检测出的gadget链条分为三种：

**Magic Method Chains:** 是指以魔法方法作为 PM的gadget 链。这类链的核心特征的是：PM 调用节点为 PHP 内置的魔法方法（如call()、toString()、__get()等），且魔法方法的触发源于 “非预期场景”—— 即开发者未主动调用魔法方法，而是因 “方法 / 属性不存在、对象类型不匹配” 等场景，导致魔法方法被自动触发。

**Possible Call Chains:** 是指不包含魔法方法调用，但存在 PM 调用节点的 gadget 链。这类链的核心特征是：PM 为普通的多态方法（即同一方法名在不同类中有不同实现），且 PM 的调用依赖 “攻击者对可控对象的类型篡改”—— 开发者预期调用某类的方法实现，攻击者通过反序列化将对象篡改为另一类，导致非预期的方法实现被执行。

**Vanilla Chains:** 是指从入口方法到 sink 函数的执行链中，不包含任何 PM 调用节点的 gadget 链。这类链的核心特征是：链结构最短，无中间 PM 节点，入口方法直接调用 sink 函数；漏洞根源并非 “方法调用非预期”，而是 “入口类可被攻击者直接反序列化，且入口方法直接触发 sink 函数.



**Magic Method Chains patch**: 能直接生成 “限制性补丁”。这类链的 PM 是魔法方法（比如call ()、toString ()），漏洞根源是 “调用了不存在的方法 / 属性，触发非预期魔法方法”—— 所以补丁很好加，就是提前 “验身份”，不符合就终止执行。
   论文里给了 8 条预定义规则，比如：触发call () 的链（像 Listing 2 里调用不存在的 getLog ()）：加个if(!method_exists($this->logger, 'getLog')) { die(); }—— 先查对象有没有这个方法，没有就直接终止，不让call () 触发；这种补丁叫 “限制性补丁”，只加一行条件判断，不改原有逻辑。

![image-20250918095515082](C:\Users\10749\AppData\Roaming\Typora\typora-user-images\image-20250918095515082.png)

  剩下两种这篇工作认为修复方式可能会影响正常功能，只是为开发者生成补丁建议。一般就是限制反序列化入口，加个白名单或者在反序列化后加一个wakeup函数来清除攻击者可控的字段。



evaluation略过了，偷个懒。
