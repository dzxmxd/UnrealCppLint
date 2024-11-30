# 修正说明

[Unreal 代码规范](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/epic-cplusplus-coding-standard-for-unreal-engine)
[Google 代码规范](https://zh-google-styleguide.readthedocs.io/en/latest/index.html)

## 一、新增说明

### 1. 花括号单独新行

> 大括号格式必须一致。在Epic的传统做法中，大括号固定被放在新行。请遵循此格式。[(链接)](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/epic-cplusplus-coding-standard-for-unreal-engine#%E5%A4%A7%E6%8B%AC%E5%8F%B7)

添加花括号必须新行的检测，同时删除了原始对左花括号必须在控制语句末尾的检测。
这里跳过了 lambda 表达式与 花括号初始化的检测。

```Python
  # Unreal: Epic has a long standing usage pattern of putting braces on a new line.
  if '{' in line and not re.match(r'^\s*{', line):
    # ignore lambda and brace initialization
    if (not re.search(r'^[^{};]*\[[^\[\]]*\][^{}]*\{[^{}\n\r]*\}', line) and
        not re.search(r'\{[^{}]*\}', line)):
      error(filename, linenum, 'whitespace/braces', 4,
            '{ should always be on a new line')
```

### 2. 单语句执行块需要大括号

> 固定在单语句块中使用大括号。[(链接)](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/epic-cplusplus-coding-standard-for-unreal-engine#%E5%A4%A7%E6%8B%AC%E5%8F%B7)

添加控制语句下一行必须使用大括号的检测。
这里需要考虑并处理条件语句可能会跨越多行的情况。
同时删除了 { 前需要一个空格的检测。

```Python
  # Unreal: Always include braces in single-statement blocks.
  if re.search(r'^\s*(if|for|while|else if)\s*\(.*\)\s*|^\s*else\s*$', line):
    control_line = line.strip()
    control_line_num = linenum
    # Unreal: Control statement conditions may span multiple lines.
    while control_line.count('(') != control_line.count(')') and control_line_num + 1 < len(clean_lines.elided):
      control_line_num += 1
      control_line += ' ' + clean_lines.elided[control_line_num].strip()
  
    next_line = clean_lines.elided[control_line_num + 1].strip() if control_line_num + 1 < len(clean_lines.elided) else ''
    if IsBlankLine(next_line):
      error(filename, control_line_num + 1, 'whitespace/blank_line', 2,
            'Redundant blank line after control statement')
    elif not re.match(r'^\s*{', next_line) and not re.match(r'.*{\s*$', control_line):
      error(filename, control_line_num + 1, 'readability/braces', 4,
            'Single statement blocks should use braces')
```

### 3. Else 需要在新行

> if-else语句中的所有执行块都应该使用大括号。此举是为防止编辑时出错——未使用大括号时，可能会意外地将另一行加入if块中。多余行不受if表达式控制，会成为较差代码。条件编译的项目导致if/else语句中断时，也会造成不良结果。因此务必使用大括号。([链接](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/epic-cplusplus-coding-standard-for-unreal-engine#if-else))

第 2 项中的检测包含了必须使用大括号的逻辑，这里额外检测 else 不会在上一行的末尾，始终在新行开始。
同时删除了 } 与 else 的同行检测，以及二者之间需要一个空格的检测。

```Python
  # Unreal: Make sure else in a new line.
  if re.search(r'}[\s]*else', line):
    error(filename, linenum, 'whitespace/braces', 5,
          'else should always be on a new line')
```

### 4. Tab 缩进检测

> 以下为代码缩进的标准。
>
> * 通过执行块缩进代码。
> * 在行的起始使用制表符，而非空格将制表符设为4字符。有时则需要使用空格，以便忽略制表符的空格数保持代码对齐。例如：以无制表符字符对齐代码。[(链接)](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/epic-cplusplus-coding-standard-for-unreal-engine#%E5%88%B6%E8%A1%A8%E7%AC%A6%E5%92%8C%E7%BC%A9%E8%BF%9B)

检测是否使用 Tab 缩进，这里没考虑规范中的特例情况，平时业务开发中应当不太需要考虑。
同时删除了原 tab 缩进与非 2、4 个空格的检测。

```Python
  # Unreal: Use tabs, not spaces, for whitespace at the beginning of a line.
  if re.match(r'^[ \t]* ', line):
    error(filename, linenum, 'whitespace/indent', 1,
          'Use tab for indentation instead of spaces.')
```

### 5. 单行大括号匹配时空格检测

UE 中存在很多的函数在一行完成，比如 Set、Get 方法、构造、析构等，这里我们保留一个单行匹配时 { 前需要空格的检测。

```Python
  # Unreal: We only detect spaces when matching single line braces
  # in a new line we don't care weather the previous line end with a spaces.
  match = re.match(r'^(.*[^ ({>\s]){.*}', line)
```

### 6. 代码注释间空格检测

UE 的实践中对于同行的代码注释，往往只在代码注释间存在一个空格，这里我们将原始的两个空格检测修正为一个。

```Python
  commentpos = line.find('//')
  if commentpos != -1:
    # Unreal: Changed the two space detections between code and comments to one.
    if re.sub(r'\\.', '', line[0:commentpos]).count('"') % 2 == 0:
      # Allow one space for new scopes, two spaces otherwise:
      if (not (re.match(r'^.*{ *//', line) and next_line_start == commentpos) and
          ((commentpos >= 1 and
            line[commentpos-1] not in string.whitespace))):
        error(filename, linenum, 'whitespace/comments', 2,
              'At least one spaces is best between code and comments')
```

### 7. C 风格转换的两个误判

当在 lambda 表达式或函数定义未命名形参时，会误判为 C 风格的转换，这里避免掉两种报错。

```Python
  # Unreal: Ignore unnamed input parameters
  # like: void UActorModifierCoreStack::OnActorDestroyed(AActor*)
  if re.search(r'\b[a-zA-Z_]\w*\s*(::\s*[a-zA-Z_]\w*)?\s*\([^)]*\)\s*$', line):
    return False
  
  # Unreal: Ignore lambda event binding
  # like: FEditorDelegates::BeginPIE.AddLambda([](bool)
  lambda_pattern = r'\[.*\]\s*\([^)]*\)\s*\{?'
  if re.search(lambda_pattern, line):
    return False
```

### 8. 检测控制语句后多余的空行

第 2 项检测中添加了此空行检测，没有时会将控制语句后存在空行的案例误报为控制语句没有使用大括号，故添加此逻辑。

```python
    if IsBlankLine(next_line):
      error(filename, control_line_num + 1, 'whitespace/blank_line', 2,
            'Redundant blank line after control statement')
    elif not re.match(r'^\s*{', next_line) and not re.match(r'.*{\s*$', control_line):
      error(filename, control_line_num + 1, 'readability/braces', 4,
            'Single statement blocks should use braces')
```

## 二、删除说明

部分与添加逻辑冲突的上面有部分描述，这里整理一下其他类型的。

### 1. 非常量引用入参的检测

在 UE 代码的实践中，大量使用了非常量引用作为出入参，以在函数内修改相关引用变量的值，并往往在命名形参时使用 Out 来标识，因此移除了这一检测 (当然也可以 filter -runtime/references 来做)。

添加注释了如下内容：

```Python
  # Unreal: In current practice, non-const reference parameters are widely used.
  # CheckForNonConstReference(filename, clean_lines, line, nesting_state, error)
```

### 2. 类成员访问修饰符前的空格检测

在 UE 代码实践中，类成员访问修饰符往往不会额外对齐，因此移除了这一检测。

删除了如下内容：

```Python
    # Update access control if we are inside a class/struct
    if self.stack and isinstance(self.stack[-1], _ClassInfo):
      classinfo = self.stack[-1]
      access_match = re.match(
          r'^(.*)\b(public|private|protected|signals)(\s+(?:slots\s*)?)?'
          r':(?:[^:]|$)',
          line)
      if access_match:
        classinfo.access = access_match.group(2)

        # Check that access keywords are indented +1 space.  Skip this
        # check if the keywords are not preceded by whitespaces.
        indent = access_match.group(1)
        if (len(indent) != classinfo.class_indent + 1 and
            re.match(r'^\s*$', indent)):
          if classinfo.is_struct:
            parent = 'struct ' + classinfo.name
          else:
            parent = 'class ' + classinfo.name
          slots = ''
          if access_match.group(3):
            slots = access_match.group(3)
          error(filename, linenum, 'whitespace/indent', 3,
                f'{access_match.group(2)}{slots}:'
                f' should be indented +1 space inside {parent}')
```

### 3. override 与 final 多余的检测

> override and final
> These keywords are valid for use, and their use is strongly encouraged. There might be many places where these have been omitted, but they will be fixed over time. [(链接，中文翻译有点问题，看英文文档吧)](https://dev.epicgames.com/documentation/en-us/unreal-engine/epic-cplusplus-coding-standard-for-unreal-engine#overrideandfinal)

UE 推荐使用此类关键字，并且说明了目前代码中缺少的部分也会逐渐修正。
因此移除了多余的函数关键字的检测。

注释了如下内容：

```Python
  # Unreal: The override and final keywords are valid for use, and their use is strongly encouraged.
  # CheckRedundantVirtual(filename, clean_lines, line, error)
  # CheckRedundantOverrideOrFinal(filename, clean_lines, line, error)
```

### 4. 命名空间不应缩进的检测

在 UE 代码实践中，命名空间与类规则类似，并没有不应缩进的实践，因此移除了这一检测

注释了如下内容：

```Python
  # Unreal: The existing practice uses indentation in the namespace, so comment here
  # CheckForNamespaceIndentation(filename, nesting_state, clean_lines, line, error)
```

### 5. 新增中涉及到的

原生 cpplint 检测是否有使用 tab 缩进，以及是否存在非 2、4 个空格缩进的情况，移除这一检测。

删除了如下内容：

```Python
  if line.find('\t') != -1:
    error(filename, linenum, 'whitespace/tab', 1,
          'Tab found; better to use spaces')

  # One or three blank spaces at the beginning of the line is weird; it's
  # hard to reconcile that with 2-space indents.
  # NOTE: here are the conditions rob pike used for his tests.  Mine aren't
  # as sophisticated, but it may be worth becoming so:  RLENGTH==initial_spaces
  # if(RLENGTH > 20) complain = 0;
  # if(match($0, " +(error|private|public|protected):")) complain = 0;
  # if(match(prev, "&& *$")) complain = 0;
  # if(match(prev, "\\|\\| *$")) complain = 0;
  # if(match(prev, "[\",=><] *$")) complain = 0;
  # if(match($0, " <<")) complain = 0;
  # if(match(prev, " +for \\(")) complain = 0;
  # if(prevodd && match(prevprev, " +for \\(")) complain = 0;
  scope_or_label_pattern = r'\s*(?:public|private|protected|signals)(?:\s+(?:slots\s*)?)?:\s*\\?$'
  classinfo = nesting_state.InnermostClass()
  initial_spaces = 0
  while initial_spaces < len(line) and line[initial_spaces] == ' ':
    initial_spaces += 1
  # There are certain situations we allow one space, notably for
  # section labels, and also lines containing multi-line raw strings.
  # We also don't check for lines that look like continuation lines
  # (of lines ending in double quotes, commas, equals, or angle brackets)
  # because the rules for how to indent those are non-trivial.
  if (not re.search(r'[",=><] *$', prev) and
      (initial_spaces == 1 or initial_spaces == 3) and
      not re.match(scope_or_label_pattern, cleansed_line) and
      not (clean_lines.raw_lines[linenum] != line and
           re.match(r'^\s*""', line))):
    error(filename, linenum, 'whitespace/indent', 3,
          'Weird number of spaces at line-start.  '
          'Are you using a 2-space indent?')
```

移除 } 与 else 间必须存在空格的检测。

删除了如下内容：

```Python
  # Make sure '} else {' has spaces.
  if re.search(r'}else', line):
    error(filename, linenum, 'whitespace/braces', 5,
          'Missing space before else')
```

删除 } 与 else 同行、} else [if] { 左右括号必须匹配的检测。

删除了如下内容：

```Python
  # An else clause should be on the same line as the preceding closing brace.
  if last_wrong := re.match(r'\s*else\b\s*(?:if\b|\{|$)', line):
    prevline = GetPreviousNonBlankLine(clean_lines, linenum)[0]
    if re.match(r'\s*}\s*$', prevline):
      error(filename, linenum, 'whitespace/newline', 4,
            'An else should appear on the same line as the preceding }')
    else:
      last_wrong = False

  # If braces come on one side of an else, they should be on both.
  # However, we have to worry about "else if" that spans multiple lines!
  if re.search(r'else if\s*\(', line):       # could be multi-line if
    brace_on_left = bool(re.search(r'}\s*else if\s*\(', line))
    # find the ( after the if
    pos = line.find('else if')
    pos = line.find('(', pos)
    if pos > 0:
      (endline, _, endpos) = CloseExpression(clean_lines, linenum, pos)
      brace_on_right = endline[endpos:].find('{') != -1
      if brace_on_left != brace_on_right:    # must be brace after if
        error(filename, linenum, 'readability/braces', 5,
              'If an else has a brace on one side, it should have it on both')
  # Prevent detection if statement has { and we detected an improper newline after }
  elif re.search(r'}\s*else[^{]*$', line) or (re.match(r'[^}]*else\s*{', line) and not last_wrong):
    error(filename, linenum, 'readability/braces', 5,
          'If an else has a brace on one side, it should have it on both')
```

## 三、其他说明

原生 cpplint 的功能基本没有调整，只是将相关的规范改为了 UE 的标准规范
有一些规则原则上也可以删除掉，但笔者这里没有考虑处理，业务如果需要，可以自行修改或使用 filter 筛选掉，比如 public 后不能跟空行、右小括号不换行、meta = 等于前后要加空格这种，很多引擎代码中也没有统一的规范，由项目、业务去决定如何处理吧。一个项目组的代码规范，由项目组自己拍板就可以了，核心还是要保证代码风格的一致性、可读性，以期更高的可维护性。

后续可能也会考虑添加一些 UE 独有的规范，就先放到 TODO 里了，目前基本足够使用了。

## 附：

[Python3 正则表达式](https://www.runoob.com/python3/python3-reg-expressions.html)

| 模式         | 描述                                                                                                                                                                      |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ^            | 匹配字符串的开头                                                                                                                                                          |
| $            | 匹配字符串的末尾。                                                                                                                                                        |
| .            | 匹配任意字符，除了换行符，当re.DOTALL标记被指定时，则可以匹配包括换行符的任意字符。                                                                                       |
| [...]        | 用来匹配所包含的任意一个字符，例如 [amk] 匹配 'a'，'m'或'k'                                                                                                               |
| [^...]       | 不在[]中的字符：[^abc] 匹配除了a,b,c之外的字符。                                                                                                                          |
| re*          | 匹配0个或多个的表达式。                                                                                                                                                   |
| re+          | 匹配1个或多个的表达式。                                                                                                                                                   |
| re?          | 匹配0个或1个由前面的正则表达式定义的片段，非贪婪方式                                                                                                                      |
| re{ n}       | 匹配n个前面表达式。例如，"o{2}"不能匹配"Bob"中的"o"，但是能匹配"food"中的两个o。                                                                                          |
| re{ n,}      | 精确匹配n个前面表达式。例如，"o{2,}"不能匹配"Bob"中的"o"，但能匹配"foooood"中的所有o。"o{1,}"等价于"o+"。"o{0,}"则等价于"o*"。                                            |
| re{ n, m}    | 匹配 n 到 m 次由前面的正则表达式定义的片段，贪婪方式                                                                                                                      |
| a            | b                                                                                                                                                                         |
| (re)         | 匹配括号内的表达式，也表示一个组                                                                                                                                          |
| (?imx)       | 正则表达式包含三种可选标志：i, m, 或 x 。只影响括号中的区域。                                                                                                             |
| (?-imx)      | 正则表达式关闭 i, m, 或 x 可选标志。只影响括号中的区域。                                                                                                                  |
| (?: re)      | 类似 (...), 但是不表示一个组                                                                                                                                              |
| (?imx: re)   | 在括号中使用i, m, 或 x 可选标志                                                                                                                                           |
| (?-imx: re)  | 在括号中不使用i, m, 或 x 可选标志                                                                                                                                         |
| (?#...)      | 注释.                                                                                                                                                                     |
| (?= re)      | 前向肯定界定符。如果所含正则表达式，以 ... 表示，在当前位置成功匹配时成功，否则失败。但一旦所含表达式已经尝试，匹配引擎根本没有提高；模式的剩余部分还要尝试界定符的右边。 |
| (?! re)      | 前向否定界定符。与肯定界定符相反；当所含表达式不能在字符串当前位置匹配时成功。                                                                                            |
| (?> re)      | 匹配的独立模式，省去回溯。                                                                                                                                                |
| \w           | 匹配数字字母下划线                                                                                                                                                        |
| \W           | 匹配非数字字母下划线                                                                                                                                                      |
| \s           | 匹配任意空白字符，等价于 [\t\n\r\f]。                                                                                                                                     |
| \S           | 匹配任意非空字符                                                                                                                                                          |
| \d           | 匹配任意数字，等价于 [0-9]。                                                                                                                                              |
| \D           | 匹配任意非数字                                                                                                                                                            |
| \A           | 匹配字符串开始                                                                                                                                                            |
| \Z           | 匹配字符串结束，如果是存在换行，只匹配到换行前的结束字符串。                                                                                                              |
| \z           | 匹配字符串结束                                                                                                                                                            |
| \G           | 匹配最后匹配完成的位置。                                                                                                                                                  |
| \b           | 匹配一个单词边界，也就是指单词和空格间的位置。例如， 'er\b' 可以匹配"never" 中的 'er'，但不能匹配 "verb" 中的 'er'。                                                      |
| \B           | 匹配非单词边界。'er\B' 能匹配 "verb" 中的 'er'，但不能匹配 "never" 中的 'er'。                                                                                            |
| \n, \t, 等。 | 匹配一个换行符。匹配一个制表符, 等                                                                                                                                        |
| \1...\9      | 匹配第n个分组的内容。                                                                                                                                                     |
| \10          | 匹配第n个分组的内容，如果它经匹配。否则指的是八进制字符码的表达式。                                                                                                       |
