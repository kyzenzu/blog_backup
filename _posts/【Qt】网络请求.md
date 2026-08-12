---
title: 【Qt】网络请求
date: 2026-08-09 16:42:46
tags:
  - C/C++
  - Qt
---

根据这段代码，我为您总结Qt中网络请求的完整流程：

## 1. **初始化网络管理器**
```cpp
// 创建QNetworkAccessManager实例
networkAccessManager = new QNetworkAccessManager(this);
// 连接finished信号，处理响应
connect(networkAccessManager, SIGNAL(finished(QNetworkReply*)),
        this, SLOT(slotNetworkRequestFinished(QNetworkReply*)));
```

## 2. **发起GET请求**
```cpp
void MainWindow::getWeatherInfo(QNetworkAccessManager* manager) {
    // 构造完整的URL
    QString citycode = this->weatherTool[this->city];
    QUrl url(this->url + citycode);  // url = "http://t.weather.itboy.net/api/weather/city/" + citycode
    
    // 发送GET请求
    manager->get(QNetworkRequest(url));
}
```
**GET请求关键步骤：**
- 创建`QUrl`对象
- 创建`QNetworkRequest(url)`
- 调用`manager->get(QNetworkRequest)`

## 3. **接收响应（finished信号槽）**
```cpp
void MainWindow::slotNetworkRequestFinished(QNetworkReply* reply)
{
    // ① 检查HTTP状态码
    QVariant status_code = reply->attribute(QNetworkRequest::HttpStatusCodeAttribute);
    
    // ② 检查错误
    if (reply->error() != QNetworkReply::NoError || status_code != 200) {
        QMessageBox::warning(this, "错误", "天气：请求数据错误，检查网络连接", QMessageBox::Ok);
        return;
    }

    // ③ 读取数据
    QByteArray jsonBytes = reply->readAll();
    
    // ④ 解析数据
    parseJson(jsonBytes);
    
    // ⑤ 更新UI
    updateUIData();
}
```

## 4. **POST请求示例**（虽然代码中没有，但补充如下）
```cpp
void MainWindow::sendPostRequest() {
    QNetworkRequest request;
    request.setUrl(QUrl("http://example.com/api"));
    request.setHeader(QNetworkRequest::ContentTypeHeader, "application/json");
    
    // 准备POST数据
    QJsonObject json;
    json["key"] = "value";
    QByteArray postData = QJsonDocument(json).toJson();
    
    // 发送POST请求
    manager->post(request, postData);
}
```

## 核心流程总结

| 步骤            | 操作                                             | 关键类/方法                                                  |
| --------------- | ------------------------------------------------ | ------------------------------------------------------------ |
| **1. 初始化**   | 创建QNetworkAccessManager                        | `new QNetworkAccessManager(this)`                            |
| **2. 连接信号** | 连接finished信号到槽函数                         | `connect(manager, &QNetworkAccessManager::finished, this, &YourClass::handleReply)` |
| **3. 发送请求** | GET: `manager->get()`<br>POST: `manager->post()` | 传入QNetworkRequest对象                                      |
| **4. 处理响应** | 在槽函数中处理QNetworkReply                      | • 检查错误<br>• 读取数据 `reply->readAll()`<br>• 解析数据<br>• 更新UI |

## 关键注意事项

1. **错误处理**：必须检查`reply->error()`和HTTP状态码
2. **数据读取**：使用`reply->readAll()`获取响应数据
3. **内存管理**：`QNetworkReply`会自动释放，但需要确保没有内存泄漏
4. **异步操作**：网络请求是异步的，响应在`finished`信号中处理
5. **线程安全**：所有网络操作应在主线程中进行

这个流程是Qt网络编程的标准模式，适用于所有HTTP/HTTPS请求场景。
