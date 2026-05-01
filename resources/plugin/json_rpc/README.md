## 功能

将包括 typora-plugin 所有功能在内的 Typora 的一切能力通过 `json-rpc` 的形式暴露出去，以供 **外部操纵 Typora**。



## 如何使用

1. 启用 json_rpc 插件。当运行 Typora 后，内部就会自动运行一个 json-rpc-server。
2. 写一个 json-rpc-client，与 Typora 交互。

以 Node.js 和 Python 为例。



### Node.js

```javascript
const rpc = require("your_path_to_typora_dir/plugin/json_rpc/node-json-rpc.js")

const initRPC = async (options) => {
    const rawClient = new rpc.Client(options)
    const request = (arg) => new Promise((resolve, reject) => {
        rawClient.call(arg, (err, resp) => {
            const error = err || resp?.error
            if (error) reject(error)
            else resolve(resp?.result)
        })
    })

    const msg = await request({ method: "ping", params: [] })
    if (msg !== "pong from typora-plugin") {
        throw new Error("RPC Auth failed")
    }

    return {
        call: (method, params = []) => request({ method, params }),
        eval: (code) => request({ method: "eval", params: [code] }),
        invoke: (plugin, fn, ...args) => request({ method: "invokePlugin", params: [plugin, fn, ...args] }),
    }
}

async function main() {
    const client = await initRPC({
        port: 5080,
        host: "127.0.0.1",
        path: "/",
        strict: false,
    })
    await client.eval("console.log('hello world')")
    console.log(await client.eval("document.title")) // Typora
    console.log(await client.eval("__plugin_utils__.typoraVersion")) // 1.9.4

    try {
        await client.eval("no_such_variable")
    } catch (err) {
        console.log(err) // { code: 500, message: 'ReferenceError: no_such_variable is not defined' }
    }

    await client.invoke("search_multi", "call")

    try {
        await client.invoke("no_such_plugin", "no_such_method")
    } catch (err) {
        console.log(err) // { code: 404, message: "No such method 'no_such_method' in plugin 'no_such_plugin'" }
    }
}

main()
```



### Python

```python
import requests

url = "http://localhost:5080"


def _call(method, *params):
    return requests.post(url, json={ "method": method, "params": params, "jsonrpc": "2.0" }).json().get("result")


def test():
    assert _call("ping") == "pong from typora-plugin"


def invoke_plugin(plugin, fn, *args):
    _call("invokePlugin", plugin, fn, *args)


def eval_typora(code):
    _call("eval", code)


if __name__ == "__main__":
    # 测试
    test()
    # 切换只读模式
    invoke_plugin("read_only", "call")
    # 资源管理器打开第一个标签页
    invoke_plugin("window_tab", "showInFinder", 0)
    # 执行alert
    eval_typora("alert('this is test')")
```



## API

### ping

- 功能：验证功能是否打通，成功时返回 `pong from typora-plugin`
- 参数：无

### invokePlugin

- 功能：调用 typora-plugin 能力
- 参数：
  - plugin：插件名称
  - fn：插件方法
  - args：插件方法参数

### eval

- 功能：执行 eval()
- 参数：
  - code：需要执行的字符串

