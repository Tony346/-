​      最近AIGC技术的大热，市面上也出现了许多类似生产时AI，其中有一大特色就是对话的输出结果是流式的出现，要成呈现出这种效果，最主要的就是要利用SSE技术，下面我就通过一个案例，使用Express和React来为大家进行演示如何在项目中实现这种效果。

效果如下

![](https://p3.ssl.qhimg.com/t01ad7f9a5b44cc3691.gif)

## 使用Express发送流式输出

我们后端使用Express来发送数据

```javascript
const express = require('express')
const app = express()
const port = 3000
//允许跨域
app.all('*', function (req, res, next) {
    res.setHeader("Access-Control-Allow-Origin", "*");
    next();
});

app.get('/sse', (req, res) => {
    const str = 'hello word!'
    // 设置 SSE 相关的响应头
    res.setHeader('Content-Type', 'text/event-stream;charset=utf-8');
    res.setHeader('Cache-Control', 'no-cache');
    res.setHeader('Connection', 'keep-alive');
    let index = 0
    const timer = setInterval(() => {
        if (index < str.length) {
            res.write("data: " + JSON.stringify({ content: str[index] }));
            index++
        } else {
            // 当所有数据都发送完毕时，结束响应
            clearInterval(timer); // 停止定时器
            res.end();
        }

    }, 100);
});

app.listen(port, () => {
    console.log(`Example app listening on port ${port}`)
})
```

​       因为我们是作为简单案例讲解，就直接先添加一个通用的中间件，来解决跨域问题。sse传输，我们需要在响应头中将Content-Type和Connection都设置为对应的字段，以此来保证，我们的响应能够被前端正确的处理。我们通过定时器，每次读取一个字符，来模拟每次响应的结果，读取完所有字符后，就将响应停止。

​      注意我们发送的数据格式要以data: 开头，因为这是SSE规范的一部分。

1. **标识行类型**：以 `data:` 开头的行告诉客户端这是一个数据行，而不是其他类型的事件，比如注释或者消息标识。这样客户端可以正确地解析和处理接收到的数据。
2. **易于解析**：SSE 是一个文本协议，因此易于解析。以 `data:` 开头的行让客户端能够轻松地区分和提取事件数据部分，因为它们遵循明确的格式。
3. **浏览器支持**：大多数现代浏览器支持 SSE 协议，可以正确地处理以 `data:` 开头的行。这使得在客户端上使用 SSE 变得更加容易，因为浏览器会自动解析 SSE 响应中的数据。
4. **规范要求**：SSE 规范要求以 `data:` 开头的行用于传输数据，这是根据规范的定义来执行的。

## 使用React对数据进行处理

```javascript
import { useEffect, useState } from "react";
import "./App.scss";

function App() {
  const [chatText, setChatText] = useState("");

  const getRes = async () => {
    try {
      const res = await fetch("http://localhost:3000/sse", {
        method: "get",
      });
      const reader = res.body?.getReader();
      let text = "";
      while (reader) {
        const { value, done } = await reader.read();
        const chars = new TextDecoder().decode(value);
        if (done) {
          break;
        }
        const dataArray = chars.trim().split("\n\n");
        const jsonObjects = dataArray.map((data) => {
          const jsonString = data.substring("data: ".length);
          return JSON.parse(jsonString);
        });
        jsonObjects.forEach((item) => {
          text += item.content;
        });
        setChatText(text);
      }
    } catch (error) {
      console.log("error", error);
    }
  };
  useEffect(() => {
    getRes();
  }, []);
  return <div>{chatText}</div>;
}

```

​       传统的请求方式主要依赖于使用Ajax来向后端发起请求，这种方式有一个明显的特点：一次请求只能接收一次响应数据，请求完成后即结束。然而，在某些场景下，我们需要持续地接收来自后端的数据更新，这时就需要摒弃传统的Ajax方式，而选择一些更适合实现实时数据更新的方法。

​      最终关注到了两个主要的API：EventSource和Fetch，它们分别具有不同的优势和用途。于是我们分别对两个api做了对比。

| 特性     | EventSource                                                  | fetch API                                                    |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 兼容性   | 广泛支持，包括Internet Explorer 8及更高版本                  | 在较新的浏览器中得到支持，不完全支持Internet Explorer        |
| 数据格式 | 只支持服务器发送的文本数据，自动转换为文本                   | 可以获取包括文本、JSON、Blob等在内的各种数据格式             |
| 错误处理 | 自动尝试重新连接，可以监听'error'事件来处理错误              | 没有内置的重试机制，需要手动处理错误并可能需要进行重试       |
| 流式处理 | 支持简单处理服务器发送的流式数据                             | 不直接支持流式处理，但可以使用Response对象的body属性获取流式接口 |
| CORS问题 | 受同源策略限制，除非服务器配置了适当的CORS头，否则无法跨源加载 | 不受同源策略限制，可以跨源请求数据，但需要服务器配置适当的CORS头 |
| 灵活性   | 只能发送GET请求，拼接字符串传参                              | 可以发起任意类型请求。传参灵活                               |

​     在使用fecth的过程中，我们还需要注意getReader这个方法， `getReader` 方法是用于处理响应体（response body）的一种方式，它返回一个可用于异步读取响应数据的 ReadableStreamDefaultReader 对象。这个方法通常用于处理大型响应或流式数据，以便在数据逐步到达时逐步处理它们，而不是一次性将整个响应数据加载到内存中。

​     通过read方法，读取stream的返回值，我们可以从返回值中解构出value和done这个两个值，但是value值是Uint8Array，我们需要将其转化成UTF-8的字符。done值主要是来区分是否读取完毕，读取中，done会返回false，所有内容都读取完成后，就会返回true，我们可以done的值，来确定响应是否结束。

​     前端处理SSE时，我们还需要注意数据格式，后端返回的是数据格式是以data开头的字符串。而我们所需要的内容是content所对应的值，我们就需要采取一些方法，将其值获取出来。

- 使用正则表达式可以轻松地提取出content数据部分`{"content":"克服"}`，但是后期维护性和代码可读性上较低

- 使用JSON.parse，需要先将字符串转化为标准的json字符串，然后转化为对象，这样进行处理的话，我们能更加灵活的进行操作，但是如果如果数据不符合JSON格式，会抛出解析错误，这就要求我们在代码中进行适当的错误处理，以防止应用崩溃。

获取的内容后，我们只需要每次将字符串进行拼接，然后重新渲染页面，就能实现打字机效果了。

