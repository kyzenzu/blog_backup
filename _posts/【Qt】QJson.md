---
title: 【Qt】QJson
date: 2026-08-09 15:53:35
tags:
  - C/C++
  - Qt
---

# Qt中JSON的完整使用指南

## 一、设计动机：为什么Qt要这样封装JSON？

### 1.1 JSON的数据模型特征

JSON本质上是一种**树形结构**，其数据模型可抽象为：
- 一个**值**（Value）可以是：
  - **基本类型**：`string`、`number`、`boolean`、`null`
  - **复合类型**：`object`（键值对集合）、`array`（值的有序列表）

> **关键理解**：JSON语法中，整个数据就是一个值。以下都是合法的JSON：
> ```json
> 1
> ```
> ```json
> true
> ```
> ```json
> "hello"
> ```

这种**递归自包含**的结构意味着：任何JSON数据，无论多复杂，都可以用 **"值"** 这个概念统一描述。

### 1.2 Qt的设计哲学

Qt对JSON的封装遵循几个核心原则：

| 原则           | 体现                                                         |
| -------------- | ------------------------------------------------------------ |
| **类型统一**   | 用`QJsonValue`统一表示所有可能的JSON值类型                   |
| **显式转换**   | 所有类型转换必须显式调用（如`.toString()`），不进行隐式自动转换 |
| **错误可恢复** | 解析错误通过`QJsonParseError`返回，而非抛出异常              |
| **内存效率**   | 使用隐式共享（Implicit Sharing），多个文档可共享同一份数据   |

### 1.3 核心类职责

| 类                | 职责                               |
| ----------------- | ---------------------------------- |
| `QJsonDocument`   | **入口**，持有解析后的数据根节点   |
| `QJsonValue`      | **胶水层**，统一表示所有JSON值类型 |
| `QJsonObject`     | 表示JSON对象（键值对集合）         |
| `QJsonArray`      | 表示JSON数组（值的有序列表）       |
| `QJsonParseError` | 记录解析错误信息                   |

---

## 二、标准使用流程

### 步骤1：包含头文件

```cpp
#include <QJsonDocument>
#include <QJsonValue>
#include <QJsonObject>
#include <QJsonArray>
#include <QJsonParseError>
```

### 步骤2：从字节流到文档对象

```cpp
QByteArray json = file.readAll();        // 原始JSON字节流
QJsonParseError err;                     // 错误信息对象
QJsonDocument jsonDoc = QJsonDocument::fromJson(json, &err);
```

**为什么必须先转`QJsonDocument`？**
- `fromJson()`会执行完整的**词法分析**和**语法分析**，验证JSON格式合法性
- 解析过程中构建内部数据结构（类似DOM树），后续所有访问基于此结构

### 步骤3：获取顶层值

```cpp
// 如果顶层是数组
QJsonArray cities = jsonDoc.array();

// 如果顶层是对象
QJsonObject root = jsonDoc.object();
```

> ⚠️ **注意**：不能同时调用`.array()`和`.object()`，因为顶层类型是确定的。若调用错误方法会返回空对象/空数组。

### 步骤4：遍历数组/对象

```cpp
// 遍历数组
for (const QJsonValue& jValue : cities) {
    QJsonObject city = jValue.toObject();
    // 处理每个元素...
}

// 遍历对象
for (const QString& key : root.keys()) {
    QJsonValue value = root.value(key);
    // 处理每个键值对...
}
```

**为什么数组元素是`QJsonValue`而不是`QJsonObject`？**
- JSON数组的元素类型不固定，可能是对象、字符串、数字甚至嵌套数组
- `QJsonValue`作为统一容器，只有调用`.toObject()`才显式转换

### 步骤5：从对象中提取字段

```cpp
QString city_code = city.value("city_code").toString();
QString city_name = city.value("city_name").toString();
int id = city.value("id").toInt();
double score = city.value("score").toDouble();
```

**`QJsonObject::value()`返回`QJsonValue`的原因：**
- 对象的值可以是任意JSON类型
- 统一用`QJsonValue`返回，由调用者根据业务逻辑转换

### 步骤6：完整代码示例（以`WeatherTool`为例）

```cpp
class WeatherTool {
    std::map<QString, QString> m_CityNameToCode;

public:
    WeatherTool() {
        // ① 读取文件
        QFile file("cities.json");
        if (!file.open(QIODevice::ReadOnly | QIODevice::Text)) {
            return;
        }
        QByteArray json = file.readAll();
        file.close();

        // ② 解析为QJsonDocument
        QJsonParseError err;
        QJsonDocument jsonDoc = QJsonDocument::fromJson(json, &err);
        if (err.error != QJsonParseError::NoError) {
            return;
        }

        // ③ 顶层是数组 → QJsonArray
        QJsonArray cities = jsonDoc.array();

        // ④ 遍历 → QJsonValue → QJsonObject
        for (const QJsonValue& jValue : cities) {
            QJsonObject city = jValue.toObject();

            // ⑤ 取值 → QJsonValue → 具体类型
            QString city_code = city.value("city_code").toString();
            QString city_name = city.value("city_name").toString();

            // ⑥ 存入容器（防御空值）
            if (!city_code.isEmpty() && !city_name.isEmpty()) {
                m_CityNameToCode[city_name] = city_code;
            }
        }
    }
};
```

---

## 三、QJsonValue的类型系统

`QJsonValue`内部维护枚举`Type`：

| 枚举值      | 对应JSON类型       | 判断方法        | 转换方法                 |
| ----------- | ------------------ | --------------- | ------------------------ |
| `Null`      | null               | `isNull()`      | —                        |
| `Bool`      | true/false         | `isBool()`      | `toBool()`               |
| `Double`    | number             | `isDouble()`    | `toDouble()` / `toInt()` |
| `String`    | string             | `isString()`    | `toString()`             |
| `Array`     | array              | `isArray()`     | `toArray()`              |
| `Object`    | object             | `isObject()`    | `toObject()`             |
| `Undefined` | 未定义（错误状态） | `isUndefined()` | —                        |

**关键行为**：
- 调用错误的转换方法（如对字符串调用`.toArray()`），会返回**空对象/空数组/默认值**，而非崩溃
- 建议先使用`isObject()`/`isArray()`等判断类型，再进行转换

---

## 四、常见陷阱与建议

| 陷阱                 | 说明                                   | 解决方案                            |
| -------------------- | -------------------------------------- | ----------------------------------- |
| **顶层类型判断错误** | 调用`.array()`时实际是对象，返回空数组 | 先用`isArray()`/`isObject()`判断    |
| **隐式共享的副作用** | 修改一个`QJsonObject`可能影响其他副本  | 如需独立修改，使用`.detach()`       |
| **数字精度丢失**     | JSON数字用double存储                   | 涉及高精度用`toDouble()`+四舍五入   |
| **字符串编码问题**   | JSON默认UTF-8                          | Qt的`QString`自动处理，无需额外转换 |
| **转换方法误用**     | 对字符串调用`.toObject()`返回空对象    | 先判断类型或确保数据结构已知        |

---

## 五、核心要点总结

> **整个QJson以`QJsonValue`为重心**——它统一表示所有JSON值类型，是连接`QJsonDocument`（入口）和具体类型（`QJsonObject`/`QJsonArray`/基础类型）的桥梁。

1. **入口**：`QJsonDocument`持有解析后的数据
2. **核心**：`QJsonValue`统一表示所有值
3. **容器**：`QJsonObject`和`QJsonArray`是值的特殊形态
4. **转换**：必须显式调用转换方法，不隐式转换
5. **错误**：通过`QJsonParseError`返回，不抛异常
