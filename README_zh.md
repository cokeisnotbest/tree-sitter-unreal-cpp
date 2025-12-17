# Tree-sitter-Unreal-Cpp Python 使用指南

## 安装前提
1. 安装Python 3.6+
2. 安装依赖包: `pip install tree-sitter`
3. 确保已编译tree-sitter-unreal-cpp语言库
4. 安装Python绑定: `pip install tree-sitter-cpp` 或本地安装

## 快速开始

### 最简示例
```python
from tree_sitter import Parser, Language
import tree_sitter_cpp

# 创建解析器并设置语言
parser = Parser()
parser.language = Language(tree_sitter_cpp.language())

# 解析Unreal C++代码
code = '''
UCLASS()
class AMyActor : public AActor {
    GENERATED_BODY()
    UPROPERTY(EditAnywhere) int32 Health;
};
'''

tree = parser.parse(bytes(code, 'utf8'))
print(f"解析成功: {tree.root_node.type}")
```

## Python 导入和使用

### 基本导入
```python
from tree_sitter import Parser, Language, Query
import tree_sitter_cpp
```

### 加载语言库和创建解析器
```python
# 创建解析器
parser = Parser()

# 方式1: 使用tree_sitter_cpp模块的语言函数（推荐）
parser.language = Language(tree_sitter_cpp.language())

# 方式2: 或直接赋值（某些版本）
# parser.language = tree_sitter_cpp.language
```

### 解析代码示例
```python
# 待解析的Unreal Engine C++代码
code = '''
UCLASS()
class AMyActor : public AActor {
    GENERATED_BODY()
    
    UPROPERTY(EditAnywhere, Category="Stats")
    int32 Health;
    
    UPROPERTY(BlueprintReadOnly)
    float Speed;
    
    UFUNCTION(BlueprintCallable, Category="Combat")
    void TakeDamage(int32 Damage);
};
'''

# 解析代码
tree = parser.parse(bytes(code, 'utf8'))
root_node = tree.root_node
```
```

## 主要接口说明

### Parser类
- **作用**: 代码解析器
- **方法**:
  - `parse(bytes_data)`: 解析字节数据，返回Tree对象
  - `language`: 属性，设置/获取语言对象
- **返回值**: Tree对象

### Tree类  
- **作用**: 语法树容器
- **属性**:
  - `root_node`: 根节点(Node对象)
  - `root_node.type`: 节点类型字符串
  - `root_node.start_point`: 起始位置(行,列)
  - `root_node.end_point`: 结束位置(行,列)
  - `root_node.children`: 子节点列表
- **返回值**: 包含语法树的对象

### Node类
- **作用**: 语法树节点
- **属性**:
  - `type`: 节点类型字符串
  - `start_byte`, `end_byte`: 字节位置
  - `start_point`, `end_point`: (行,列)位置元组
  - `children`: 子节点列表
  - `text`: 节点文本内容(字节串)
  - `child_count`: 子节点数量
  - `named_child_count`: 命名子节点数量
- **方法**:
  - `child(index)`: 获取第index个子节点
  - `named_child(index)`: 获取第index个命名子节点
  - `child_by_field_name(field_name)`: 根据字段名获取子节点
- **返回值**: 节点对象或相应数据类型

### Language类
- **作用**: 语言定义
- **构造**: `Language(callable_or_path, language_name=None)`
- **返回值**: 语言对象

### 常量
- `tree_sitter_cpp.language`: 语言加载函数
- 支持的节点类型常量(部分):
  - `'class_specifier'`: 类声明
  - `'struct_specifier'`: 结构体声明  
  - `'uproperty_macro'`: UPROPERTY宏
  - `'ufunction_macro'`: UFUNCTION宏
  - `'uclass_macro'`: UCLASS宏
  - `'preproc_def'`: 预处理定义
  - `'comment'`: 注释
  - `'ERROR'`: 解析错误节点类型

## 返回值说明
- `parser.parse()`: 返回Tree对象
- `tree.root_node`: 返回根Node对象  
- `node.children`: 返回Node对象列表
- 位置属性返回元组或整数

## 支持的Unreal语法元素

### 宏类型
- `UCLASS()` - 类声明宏
- `UPROPERTY()` - 属性声明宏  
- `UFUNCTION()` - 函数声明宏
- `USTRUCT()` - 结构体声明宏
- `UENUM()` - 枚举声明宏
- `GENERATED_BODY()` - 生成体宏
- `GENERATED_USTRUCT_BODY()` - 结构体生成体
- `MYPROJECT_API` - API导出宏

### Specifier关键字(35+)
**类修饰符**: Blueprintable, BlueprintType, Abstract, Sealed, NotBlueprintable

**属性修饰符**: 
- Edit类: EditAnywhere, EditDefaultsOnly, EditInstanceOnly
- 显示类: VisibleAnywhere, VisibleDefaultsOnly, VisibleInstanceOnly  
- 其他: Category, BlueprintReadOnly, BlueprintReadWrite, Transient, SaveGame

**函数修饰符**:
- Blueprint类: BlueprintCallable, BlueprintPure, BlueprintImplementableEvent
- 网络类: Server, Client, NetMulticast, Reliable, Unreliable
- 其他: Category, WithValidation

**元数据类型**: meta=(DisplayName="...", ToolTip="...")

## 实用示例

### 遍历所有UPROPERTY
```python
def find_uproperties(node):
    props = []
    if node.type == 'uproperty_macro':
        props.append(node.text.decode('utf8'))
    for child in node.children:
        props.extend(find_uproperties(child))
    return props

uproperties = find_uproperties(root_node)
```

### 提取类信息
```python
def find_classes(node):
    classes = []
    if node.type == 'class_specifier':
        name_node = node.child_by_field_name('name')
        if name_node:
            classes.append(name_node.text.decode('utf8'))
    for child in node.children:
        classes.extend(find_classes(child))
    return classes
```

### 查找所有Unreal宏
```python
def find_unreal_macros(node, macro_type=None):
    macros = []
    if node.type.endswith('_macro'):
        if not macro_type or macro_type in node.type:
            macros.append((node.type, node.text.decode('utf8')))
    for child in node.children:
        macros.extend(find_unreal_macros(child, macro_type))
    return macros

# 查找所有UCLASS
uclasses = find_unreal_macros(root_node, 'uclass')
# 查找所有UPROPERTY  
uprops = find_unreal_macros(root_node, 'uproperty')
```

### 使用Query API查询
```python
from tree_sitter import Query

query = Query(
    tree_sitter_cpp.language(),
    '(class_specifier (preproc_def (identifier) @class_name)) @class_def'
)
captures = query.captures(root_node)
for capture in captures:
    print(f"Found: {capture[0].text.decode('utf8')}")
```

## 命令行测试
```bash
# 激活虚拟环境
venv\\Scripts\\activate

# 运行测试脚本
python simple_test.py
```

## 编辑器集成
- **Neovim**: 配置tree-sitter解析器
- **VSCode**: 使用tree-sitter插件
- **其他**: 支持任何tree-sitter兼容编辑器

## 注意事项
1. 代码必须以UTF-8编码传入
2. 确保语言库文件路径正确
3. 复杂查询可使用tree-sitter Query API
4. 性能: 约50000行/秒解析速度