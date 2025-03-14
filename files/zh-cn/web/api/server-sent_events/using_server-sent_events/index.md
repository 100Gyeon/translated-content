---
title: 使用服务器发送事件
slug: Web/API/Server-sent_events/Using_server-sent_events
---

{{DefaultAPISidebar("Server Sent Events")}}

开发一个使用服务器发送的事件的 Web 应用程序是很容易的。你需要在服务器上的一些代码将事件流传输到 Web 应用程序，但 Web 应用程序端的事情几乎完全相同，处理任何其他类型的事件。

在 Web 应用程序中使用服务器发送事件很简单。在服务器端，只需要按照一定的格式返回事件流，在客户端中，只需要为一些事件类型绑定监听函数，和处理其他普通的事件没多大区别。

## 从服务器接受事件

服务器发送事件 API 也就是 {{domxref("EventSource")}} 接口，在你创建一个新的 {{domxref("EventSource")}} 对象的同时，你可以指定一个接受事件的 URI。例如：

```
const evtSource = new EventSource("ssedemo.php");
```

> **备注：** 从 Firefox 11 开始，`EventSource`开始支持[CORS](/zh-CN/HTTP_access_control).虽然该特性目前并不是标准，但很快会成为标准。

如果发送事件的脚本不同源，应该创建一个新的包含 URL 和 options 参数的`EventSource`对象。例如，假设客户端脚本在 example.com 上：

```js
const evtSource = new EventSource("//api.example.com/ssedemo.php", { withCredentials: true } );
```

一旦你成功初始化了一个事件源，就可以对 {{domxref("EventSource.message_event", "message")}} 事件添加一个处理函数开始监听从服务器发出的消息了：

```js
evtSource.onmessage = function(event) {
  const newElement = document.createElement("li");
  const eventList = document.getElementById("list");

  newElement.innerHTML = "message: " + event.data;
  eventList.appendChild(newElement);
}
```

上面的代码监听了那些从服务器发送来的所有没有指定事件类型的消息 (没有`event`字段的消息),然后把消息内容显示在页面文档中。

你也可以使用`addEventListener()`方法来监听其他类型的事件：

```js
evtSource.addEventListener("ping", function(event) {
  const newElement = document.createElement("li");
  const time = JSON.parse(event.data).time;
  newElement.innerHTML = "ping at " + time;
  eventList.appendChild(newElement);
});
```

这段代码也类似，只是只有在服务器发送的消息中包含一个值为"ping"的`event`字段的时候才会触发对应的处理函数，也就是将`data`字段的字段值解析为 JSON 数据，然后在页面上显示出所需要的内容。

> **警告：** 当**不通过 HTTP / 2 使用时**，SSE（server-sent events）会受到最大连接数的限制，这在打开各种选项卡时特别麻烦，因为该限制是针对每个浏览器的，并且被设置为一个非常低的数字（6）。该问题在 [Chrome](https://bugs.chromium.org/p/chromium/issues/detail?id=275955) 和 [Firefox](https://bugzilla.mozilla.org/show_bug.cgi?id=906896)中被标记为“无法解决”。此限制是针对每个浏览器 + 域的，因此这意味着您可以跨所有选项卡打开 6 个 SSE 连接到 www\.example1.com，并打开 6 个 SSE 连接到 www\.example2.com。（来自 [Stackoverflow](https://stackoverflow.com/a/5326159/1905229)）。使用 HTTP / 2 时，HTTP 同一时间内的最大连接数由服务器和客户端之间协商（默认为 100）。

## 服务器端如何发送事件流

服务器端发送的响应内容应该使用值为`text/event-stream`的 MIME 类型。每个通知以文本块形式发送，并以一对换行符结尾。有关事件流的格式的详细信息，请参见[事件流格式](#事件流格式)。

演示的{{Glossary("PHP")}}代码如下：

```php
date_default_timezone_set("America/New_York");
header("Cache-Control: no-cache");
header("Content-Type: text/event-stream");

$counter = rand(1, 10);
while (true) {
  // Every second, send a "ping" event.

  echo "event: ping\n";
  $curDate = date(DATE_ISO8601);
  echo 'data: {"time": "' . $curDate . '"}';
  echo "\n\n";

  // Send a simple message at random intervals.

  $counter--;

  if (!$counter) {
    echo 'data: This is a message at time ' . $curDate . "\n\n";
    $counter = rand(1, 10);
  }

  ob_end_flush();
  flush();
  sleep(1);
}
```

上面的代码会让服务器每隔一秒生成一个事件流并返回，其中每条消息的事件类型为"ping",数据字段都使用了 JSON 格式，数组字段中包含了每个事件流生成时的 ISO 8601 时间戳。而且会随机返回一些无事件类型的消息。

> **备注：** 您可以在 github 上找到以上代码的完整示例—参见[Simple SSE demo using PHP.](https://github.com/mdn/dom-examples/tree/master/server-sent-events)

## 错误处理

当发生错误 (例如请求超时或与[HTTP 访问控制（CORS）](/zh-CN/docs/Web/HTTP/Access_control_CORS)有关的问题), 会生成一个错误事件。您可以通过在`EventSource`对象上使用`onerror`回调来对此采取措施：

```
evtSource.onerror = function(err) {
  console.error("EventSource failed:", err);
};
```

## 关闭事件流

默认情况下，如果客户端和服务器之间的连接关闭，则连接将重新启动。可以使用`.close()`方法终止连接。

```
evtSource.close();
```

## 事件流格式

事件流仅仅是一个简单的文本数据流，文本应该使用 [UTF-8](/zh-CN/docs/Glossary/UTF-8) 格式的编码。每条消息后面都由一个空行作为分隔符。以冒号开头的行为注释行，会被忽略。

> **备注：** 注释行可以用来防止连接超时，服务器可以定期发送一条消息注释行，以保持连接不断。

每条消息是由多个字段组成的，每个字段由字段名，一个冒号，以及字段值组成。

### 字段

规范中规定了下面这些字段：

- `event`
  - : 事件类型。如果指定了该字段，则在客户端接收到该条消息时，会在当前的`EventSource`对象上触发一个事件，事件类型就是该字段的字段值，你可以使用`addEventListener() 方法在当前 EventSource`对象上监听任意类型的命名事件，如果该条消息没有`event`字段，则会触发`onmessage 属性上的事件处理函数`.
- `data`
  - : 消息的数据字段。如果该条消息包含多个`data`字段，则客户端会用换行符把它们连接成一个字符串来作为字段值。
- `id`
  - : 事件 ID，会成为当前`EventSource`对象的内部属性"最后一个事件 ID"的属性值。
- `retry`
  - : 一个整数值，指定了重新连接的时间 (单位为毫秒),如果该字段值不是整数，则会被忽略。

除了上面规定的字段名，其他所有的字段名都会被忽略。

> **备注：** 如果一行文本中不包含冒号，则整行文本会被解析成为字段名，其字段值为空。

### 例子

#### 未命名事件

下面的例子中发送了三条消息，第一条仅仅是个注释，因为它以冒号开头。第二条消息只包含了一个`data`字段，值为"some text".第三条消息包含的两个`data`字段会被解析成为一个字段，值为"another message\nwith two lines".其中每两条消息之间是以一个空行为分割符的。

```
: this is a test stream

data: some text

data: another message
data: with two lines
```

#### 命名事件

下面的事件流中包含了一些命名事件。每个事件的类型都是由`event`字段指定的，另外每个`data`字段的值可以使用 JSON 格式，当然也可以不是。

```
event: userconnect
data: {"username": "bobby", "time": "02:33:48"}

event: usermessage
data: {"username": "bobby", "time": "02:34:11", "text": "Hi everyone."}

event: userdisconnect
data: {"username": "bobby", "time": "02:34:23"}

event: usermessage
data: {"username": "sean", "time": "02:34:36", "text": "Bye, bobby."}
```

#### 混合两种事件

你可以在一个事件流中同时使用命名事件和未命名事件。

```
event: userconnect
data: {"username": "bobby", "time": "02:33:48"}

data: Here's a system message of some kind that will get used
data: to accomplish some task.

event: usermessage
data: {"username": "bobby", "time": "02:34:11", "text": "Hi everyone."}
```

## 浏览器兼容性

{{Compat}}
