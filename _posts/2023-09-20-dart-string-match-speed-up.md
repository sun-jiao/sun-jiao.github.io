---
layout: post
title: 让你的Dart字符串搜索提速一倍，无需任何算法知识。
categories: [Development]
description: 让你的Dart字符串搜索提速一倍
keywords: Flutter, 搜索, Dart
---

不需要任何算法知识，就能让你的Dart字符串搜索提速一倍，还有这种好事？

哈哈哈，标题党了，其实这件事说起来非常简单，一言以蔽之：Dart的正则搜索比字符串搜索快很多，所以直接将字符串转换成正则表达式来搜索就行了。

我们先做一个小测试：

下载微软研究院的[WikiQA语料库](https://www.microsoft.com/en-us/download/details.aspx?id=52419)，在里面进行搜索。

先导入库：
```dart
import 'dart:io';

late final String file;
```

定义一个计时器：
```dart
timer(String algorithmName, String target,
    Iterable<Match> Function(String target, String text) findFunction) {
  final start = DateTime.now().millisecondsSinceEpoch;
  final result = findFunction.call(target, file);
  final length = result.length;
  final end = DateTime.now().millisecondsSinceEpoch;
  print('$algorithmName finds $length matches, costs ${end - start} milliseconds.');
}
```
有人可能会问为什么要单独建立一个`length`变量，直接在print里面引用`${result.length}`不行吗？

还真不行，我们后面会解释。

下面进行搜索：
```dart
Future<void> main() async {
  file = await File('WikiQA').readAsString();
  final targets = [
    "newspaper editor",
    "school girl",
    "United States",
    "Chinese Civil War",
    "native speaker",
    "very interesting",
    "disappeared",
    "African Americans",
    "There are also",
    "nuclear power",
    "comprehensive astronomical",
    "First Lady of the United States"
  ];
  
  for (String target in targets) {
    print('\nSearching "$target":');
    timer('String', target, (target, text) => target.allMatches(text));
    timer('RegExp', target, (target, text) => RegExp(RegExp.escape(target)).allMatches(text));
  }
}
```

`targets`里面的字符串其实是我在语料库里面看心情随便选出来的，其实我们也可以用`math`库的`Random`类来实现一个在语料库里随机选择字符串的函数，但是我懒得写。

当然，如果有谁要觉得我是故意选出来一些正则搜得比字符串搜得快的案例来博人眼球，那倒也duck不必，因为我们后面会阅读源码来证明正则是真的快。

运行结果如下：
```dart
Searching "newspaper editor":
String finds 2 matches, costs 24 milliseconds.
RegExp finds 2 matches, costs 10 milliseconds.

Searching "school girl":
String finds 1 matches, costs 20 milliseconds.
RegExp finds 1 matches, costs 9 milliseconds.

Searching "United States":
String finds 3078 matches, costs 18 milliseconds.
RegExp finds 3078 matches, costs 9 milliseconds.

Searching "Chinese Civil War":
String finds 21 matches, costs 17 milliseconds.
RegExp finds 21 matches, costs 12 milliseconds.

Searching "native speaker":
String finds 6 matches, costs 20 milliseconds.
RegExp finds 6 matches, costs 12 milliseconds.

Searching "very interesting":
String finds 1 matches, costs 18 milliseconds.
RegExp finds 1 matches, costs 12 milliseconds.

Searching "disappeared":
String finds 9 matches, costs 19 milliseconds.
RegExp finds 9 matches, costs 9 milliseconds.

Searching "African Americans":
String finds 14 matches, costs 17 milliseconds.
RegExp finds 14 matches, costs 11 milliseconds.

Searching "There are also":
String finds 18 matches, costs 18 milliseconds.
RegExp finds 18 matches, costs 13 milliseconds.

Searching "nuclear power":
String finds 19 matches, costs 21 milliseconds.
RegExp finds 19 matches, costs 11 milliseconds.

Searching "comprehensive astronomical":
String finds 2 matches, costs 18 milliseconds.
RegExp finds 2 matches, costs 11 milliseconds.

Searching "First Lady of the United States":
String finds 9 matches, costs 17 milliseconds.
RegExp finds 9 matches, costs 13 milliseconds.
```

我们看到，最快的能将时间缩短到原来的一半还短。

我们上面的正则搜索的语句是这样写的：
```dart
RegExp(RegExp.escape(target)).allMatches(text)
```
`RegExp.escape(target)`是将正则表达式自动转义的方法，可以避免一些特殊符号被识别成正则语法。转义后的字符串用`RegExp(String text)`即可转换为正则表达式。

但是这样写仍然太麻烦了，于是我们可以将其写成一个extension，方便后续调用：
```dart
extension on String {
  Iterable<RegExpMatch> allRegexMatches(String input, [int start = 0]) =>
    RegExp(RegExp.escape(this)).allMatches(input, start);
}
```

后续只要使用`target.allRegexMatches(text)`即可。

不想看具体原理的到这里的就可以结束了。
—————————————————————————

然后我们来读源码：

`String.allMatches`这个方法位于dart-lang/sdk/lib/_internal/vm/lib/string_patch.dart中：
```dart
Iterable<Match> allMatches(String string, [int start = 0]) {
  if (start < 0 || start > string.length) {
    throw new RangeError.range(start, 0, string.length, "start");
  }
  return new _StringAllMatchesIterable(string, this, start);
}
```

显然下一步应该去找它所调用的另一个方法`_StringAllMatchesIterable`：

```dart
final class _StringAllMatchesIterable extends Iterable<Match> {
  final String _input;
  final String _pattern;
  final int _index;

  _StringAllMatchesIterable(this._input, this._pattern, this._index);

  Iterator<Match> get iterator =>
      new _StringAllMatchesIterator(_input, _pattern, _index);

  Match get first {
    int index = _input.indexOf(_pattern, _index);
    if (index >= 0) {
      return new _StringMatch(index, _input, _pattern);
    }
    throw IterableElementError.noElement();
  }
}

final class _StringAllMatchesIterator implements Iterator<Match> {
  final String _input;
  final String _pattern;
  int _index;
  Match? _current;

  _StringAllMatchesIterator(this._input, this._pattern, this._index);

  bool moveNext() {
    if (_index + _pattern.length > _input.length) {
      _current = null;
      return false;
    }
    var index = _input.indexOf(_pattern, _index);
    if (index < 0) {
      _index = _input.length + 1;
      _current = null;
      return false;
    }
    int end = index + _pattern.length;
    _current = new _StringMatch(index, _input, _pattern);
    // Empty match, don't start at same location again.
    if (end == _index) end++;
    _index = end;
    return true;
  }

  Match get current => _current as Match;
}
```
（以上代码能看出来，allMatches返回的只是一个Iterable，搜索实际上是在这个值被访问时才真正进行的，所以上面的timer函数必须在获取结束时间之前访问一次这个Iterable，否则返回的时间差是0。）

可以看到，`moveNext`方法在搜索时会调用`_input.indexOf(_pattern, _index)`。

然后去看`indexOf`的源码：
```dart
int indexOf(Pattern pattern, [int start = 0]) {
  if ((start < 0) || (start > this.length)) {
    throw new RangeError.range(start, 0, this.length, "start");
  }
  if (pattern is String) {
    String other = pattern;
    int maxIndex = this.length - other.length;
    // TODO: Use an efficient string search (e.g. BMH).
    for (int index = start; index <= maxIndex; index++) {
      if (_substringMatches(index, other)) {
        return index;
      }
    }
    return -1;
  }
  for (int i = start; i <= this.length; i++) {
    // TODO(11276); This has quadratic behavior because matchAsPrefix tries
    // to find a later match too. Optimize matchAsPrefix to avoid this.
    if (pattern.matchAsPrefix(this, i) != null) return i;
  }
  return -1;
}
```

震惊！居然就是`for (int index = start; index <= maxIndex; index++)`爆算，那么字符串搜索这么慢也不足为奇了。

而dart的正则类`RegExp`实际上是用C++实现的。而正则表达式搜索代码`RegExp_ExecuteMatch`位于sdk/runtime/vm/compiler/asm_intrinsifier.cc。

```c++
void AsmIntrinsifier::RegExp_ExecuteMatch(Assembler* assembler,
                                          Label* normal_ir_body) {
  AsmIntrinsifier::IntrinsifyRegExpExecuteMatch(assembler, normal_ir_body,
                                                /*sticky=*/false);
}

void AsmIntrinsifier::RegExp_ExecuteMatchSticky(Assembler* assembler,
                                                Label* normal_ir_body) {
  AsmIntrinsifier::IntrinsifyRegExpExecuteMatch(assembler, normal_ir_body,
                                                /*sticky=*/true);
}
```
而它所调用的`IntrinsifyRegExpExecuteMatch`是每个处理器架构分别实现的，里面甚至还有一部分汇编代码。

<img src="https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/507283c872534111a250eaffff455478~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=532&h=599&s=54299&e=png&b=22272e" width=35%/>

以arm64实现为例，
```
void AsmIntrinsifier::IntrinsifyRegExpExecuteMatch(Assembler* assembler,
                                                   Label* normal_ir_body,
                                                   bool sticky) {
  if (FLAG_interpret_irregexp) return;

  const intptr_t kRegExpParamOffset = 2 * target::kWordSize;
  const intptr_t kStringParamOffset = 1 * target::kWordSize;
  // start_index smi is located at offset 0.

  // Incoming registers:
  // R0: Function. (Will be reloaded with the specialized matcher function.)
  // R4: Arguments descriptor. (Will be preserved.)
  // R5: Unknown. (Must be GC safe on tail call.)

  // Load the specialized function pointer into R0. Leverage the fact the
  // string CIDs as well as stored function pointers are in sequence.
  __ ldr(R2, Address(SP, kRegExpParamOffset));
  __ ldr(R1, Address(SP, kStringParamOffset));
  __ LoadClassId(R1, R1);
  __ AddImmediate(R1, -kOneByteStringCid);
#if !defined(DART_COMPRESSED_POINTERS)
  __ add(R1, R2, Operand(R1, LSL, target::kWordSizeLog2));
#else
  __ add(R1, R2, Operand(R1, LSL, target::kWordSizeLog2 - 1));
#endif
  __ LoadCompressed(FUNCTION_REG,
                    FieldAddress(R1, target::RegExp::function_offset(
                                         kOneByteStringCid, sticky)));

  // Registers are now set up for the lazy compile stub. It expects the function
  // in R0, the argument descriptor in R4, and IC-Data in R5.
  __ eor(R5, R5, Operand(R5));

  // Tail-call the function.
  __ LoadCompressed(
      CODE_REG, FieldAddress(FUNCTION_REG, target::Function::code_offset()));
  __ ldr(R1,
         FieldAddress(FUNCTION_REG, target::Function::entry_point_offset()));
  __ br(R1);
}
```

死去的记忆在攻击我……（）

单独拿出来一句让我解释，我大概能知道这是什么指令，用于把数据从内存上的某地址加载到某个寄存器之类的。

但是这么一大段让我解释它用的是什么搜索算法，首先我真的看不懂（），而且我估计掘金上应该还是有大佬的，我就不班门弄斧了。以后说不定会有大佬来讲呢，我们还是坐等吧。等等党永不为奴！

不过我们知道，C/C++/汇编效率显然会比Dart高；而且也可以猜到，这里应该也使用了高级一些的算法。

因为如果正则搜索算法也是`for(i=0;i<length;i++)`这样爆算的话，就没必要用汇编来实现，高射炮打蚊子了。

所以尽管我半懂不懂的，但是大概能下定论，Dart语言的RegExp搜索从原理上就比String搜索要快，如果有涉及到长文本的搜索，将字符串搜索转换为正则搜索肯定是能加快搜索速度的。
