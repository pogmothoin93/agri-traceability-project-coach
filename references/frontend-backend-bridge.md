# 农产品溯源 · 前端 ↔ 后端桥接层

目标链路：

```
浏览器(IHTML界面) --HTTP--> AgriTraceServer(Java) --SDK--> FISCO BCOS 节点
```

前端 `农产品溯源-前端界面.html` 里所有按钮调的都是这些东西：

| 前端调用 | 参数 | 后端做的事 |
|---------|------|-----------|
| `POST /api/deploy`   | 无 | `AgriTrace.deploy`，返回合约地址 |
| `POST /api/register` | batchId,name,origin,producer | `register` 交易 |
| `POST /api/addtrace` | batchId,operation,operator,info | `addTrace` 交易 |
| `POST /api/product`  | batchId | `getProduct` 返回 String |
| `POST /api/trace`    | batchId,index | `getTrace` 返回 String |
| `POST /api/count`    | batchId | `getTraceCount` 返回 BigInteger |

返回统一 JSON：`{"success":true,"data":"...","msg":"..."}` 或 `{"success":false,"msg":"..."}`。

---

## 1. 新建 `AgriTraceServer.java`

路径：`asset-app/src/main/java/org/fisco/bcos/asset/client/AgriTraceServer.java`

用 JDK 自带的 `com.sun.net.httpserver.HttpServer`（零外部依赖，不联网、不踩 SDK 版本坑）。
它直接复用已跑通的 `AgriTraceClient` 里的连接初始化 + 合约调用逻辑。

```java
package org.fisco.bcos.asset.client;

import com.sun.net.httpserver.HttpExchange;
import com.sun.net.httpserver.HttpHandler;
import com.sun.net.httpserver.HttpServer;

import java.io.File;
import java.io.IOException;
import java.io.OutputStream;
import java.math.BigInteger;
import java.net.InetSocketAddress;
import java.net.URLDecoder;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.Map;

import org.fisco.bcos.asset.contract.AgriTrace;
import org.fisco.bcos.sdk.BcosSDK;
import org.fisco.bcos.sdk.client.Client;
import org.fisco.bcos.sdk.crypto.keypair.CryptoKeyPair;
import org.fisco.bcos.sdk.model.TransactionReceipt;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

/**
 * 农产品溯源 · HTTP 后端桥接层
 * 用法: bash http_run.sh   然后浏览器打开 http://localhost:8080
 */
public class AgriTraceServer {

    private static final int PORT = 8080;
    // 网页文件位置：dist/web/index.html，可用 -Dagri.web 覆盖
    private static final String WEB_DIR =
            System.getProperty("agri.web", "web");

    private static Client client;
    private static CryptoKeyPair cryptoKeyPair;

    // 与 AgriTraceClient 共用同一个合约地址类（简易版：直接读 conf 里记录）
    private static String contractAddress = null;

    public static void main(String[] args) throws Exception {
        initSdk();

        HttpServer server = HttpServer.create(new InetSocketAddress(PORT), 0);
        server.createContext("/", new StaticHandler());
        server.createContext("/api/deploy",   wrap(new DeployHandler()));
        server.createContext("/api/register", wrap(new RegisterHandler()));
        server.createContext("/api/addtrace", wrap(new AddTraceHandler()));
        server.createContext("/api/product",  wrap(new ProductHandler()));
        server.createContext("/api/trace",    wrap(new TraceHandler()));
        server.createContext("/api/count",    wrap(new CountHandler()));
        server.setExecutor(java.util.concurrent.Executors.newFixedThreadPool(4));
        server.start();

        System.out.println("========================================");
        System.out.println(" 农产品溯源系统已启动");
        System.out.println(" 浏览器打开: http://localhost:" + PORT);
        System.out.println(" (按 Ctrl+C 停止)");
        System.out.println("========================================");
    }

    // ---------- SDK 初始化（与 AgriTraceClient 一致） ----------
    private static void initSdk() throws Exception {
        @SuppressWarnings("resource")
        ApplicationContext context =
                new ClassPathXmlApplicationContext("classpath:applicationContext.xml");
        BcosSDK sdk = context.getBean(BcosSDK.class);
        client = sdk.getClient(1);
        cryptoKeyPair = client.getCryptoSuite().createKeyPair();
        client.getCryptoSuite().setCryptoKeyPair(cryptoKeyPair);
        System.out.println("SDK 初始化完成, 账户地址 = " + cryptoKeyPair.getAddress());
        // 尝试从 contract.properties 读取已有的 agriTraceAddress
        try {
            java.util.Properties p = new java.util.Properties();
            java.io.InputStream in =
                    new org.springframework.core.io.ClassPathResource("contract.properties")
                            .getInputStream();
            p.load(in);
            contractAddress = p.getProperty("agriTraceAddress");
            if (contractAddress != null && !contractAddress.trim().isEmpty()) {
                System.out.println("从 conf 读到合约地址: " + contractAddress);
            }
        } catch (Exception ignore) {
            // 首次启动还没有地址，忽略
        }
    }

    private static AgriTrace loadContract() {
        if (contractAddress == null || contractAddress.trim().isEmpty()) {
            throw new IllegalStateException("尚未部署合约，请先在「① 部署合约」页点部署");
        }
        return AgriTrace.load(contractAddress, client, cryptoKeyPair);
    }

    // ---------- API 实现 ----------
    static class DeployHandler implements HttpHandler {
        public void handle(HttpExchange ex) throws IOException {
            try {
                AgriTrace c = AgriTrace.deploy(client, cryptoKeyPair);
                contractAddress = c.getContractAddress();
                recordAddr(contractAddress);
                ok(ex, contractAddress, "部署成功");
            } catch (Exception e) {
                fail(ex, e.getMessage());
            }
        }
    }

    static class RegisterHandler implements HttpHandler {
        public void handle(HttpExchange ex) throws IOException {
            try {
                Map<String, String> q = query(ex);
                AgriTrace c = loadContract();
                TransactionReceipt r = c.register(
                        q.get("batchId"), q.get("name"), q.get("origin"), q.get("producer"));
                if (!r.isStatusOK()) {
                    fail(ex, "交易失败 status=" + r.getStatus() + " " + r.getStatusMsg());
                    return;
                }
                ok(ex, "", "登记成功, batchId=" + q.get("batchId"));
            } catch (Exception e) {
                fail(ex, e.getMessage());
            }
        }
    }

    static class AddTraceHandler implements HttpHandler {
        public void handle(HttpExchange ex) throws IOException {
            try {
                Map<String, String> q = query(ex);
                AgriTrace c = loadContract();
                TransactionReceipt r = c.addTrace(
                        q.get("batchId"), q.get("operation"), q.get("operator"), q.get("info"));
                if (!r.isStatusOK()) {
                    fail(ex, "交易失败 status=" + r.getStatus() + " " + r.getStatusMsg());
                    return;
                }
                ok(ex, "", "溯源环节已上链");
            } catch (Exception e) {
                fail(ex, e.getMessage());
            }
        }
    }

    static class ProductHandler implements HttpHandler {
        public void handle(HttpExchange ex) throws IOException {
            try {
                Map<String, String> q = query(ex);
                AgriTrace c = loadContract();
                String r = c.getProduct(q.get("batchId"));
                ok(ex, r, "ok");
            } catch (Exception e) {
                fail(ex, e.getMessage());
            }
        }
    }

    static class TraceHandler implements HttpHandler {
        public void handle(HttpExchange ex) throws IOException {
            try {
                Map<String, String> q = query(ex);
                AgriTrace c = loadContract();
                int idx = Integer.parseInt(q.getOrDefault("index", "0"));
                String r = c.getTrace(q.get("batchId"), BigInteger.valueOf(idx));
                ok(ex, r, "ok");
            } catch (Exception e) {
                fail(ex, e.getMessage());
            }
        }
    }

    static class CountHandler implements HttpHandler {
        public void handle(HttpExchange ex) throws IOException {
            try {
                Map<String, String> q = query(ex);
                AgriTrace c = loadContract();
                BigInteger n = c.getTraceCount(q.get("batchId"));
                ok(ex, n.toString(), "ok");
            } catch (Exception e) {
                fail(ex, e.getMessage());
            }
        }
    }

    // ---------- 静态文件（托管网页） ----------
    static class StaticHandler implements HttpHandler {
        public void handle(HttpExchange ex) throws IOException {
            String path = ex.getRequestURI().getPath();
            if (path.equals("/") || path.isEmpty()) {
                path = "/index.html";
            }
            Path file = Paths.get(WEB_DIR, path);
            if (!Files.exists(file) || Files.isDirectory(file)) {
                byte[] msg = ("前端文件不存在: " + file.toAbsolutePath()
                        + "\n请把 index.html 放到 dist/web/ 下").getBytes(StandardCharsets.UTF_8);
                ex.sendResponseHeaders(404, msg.length);
                try (OutputStream os = ex.getResponseBody()) { os.write(msg); }
                return;
            }
            byte[] body = Files.readAllBytes(file);
            String ct = path.endsWith(".css") ? "text/css"
                    : path.endsWith(".js") ? "application/javascript"
                    : "text/html; charset=UTF-8";
            ex.getResponseHeaders().set("Content-Type", ct);
            ex.sendResponseHeaders(200, body.length);
            try (OutputStream os = ex.getResponseBody()) { os.write(body); }
        }
    }

    // ---------- 工具 ----------
    private static HttpHandler wrap(HttpHandler h) {
        return h;
    }

    private static Map<String, String> query(HttpExchange ex) throws IOException {
        Map<String, String> map = new HashMap<>();
        String raw = ex.getRequestURI().getRawQuery();
        if (raw != null) {
            for (String pair : raw.split("&")) {
                int i = pair.indexOf('=');
                if (i > 0) {
                    String k = URLDecoder.decode(pair.substring(0, i), "UTF-8");
                    String v = URLDecoder.decode(pair.substring(i + 1), "UTF-8");
                    map.put(k, v);
                }
            }
        }
        return map;
    }

    private static void ok(HttpExchange ex, String data, String msg) throws IOException {
        String json = "{\"success\":true,\"data\":\"" + esc(data)
                + "\",\"msg\":\"" + esc(msg) + "\"}";
        send(ex, json);
    }

    private static void fail(HttpExchange ex, String msg) throws IOException {
        String json = "{\"success\":false,\"msg\":\"" + esc(msg == null ? "未知错误" : msg) + "\"}";
        send(ex, json);
    }

    private static void send(HttpExchange ex, String json) throws IOException {
        byte[] body = json.getBytes(StandardCharsets.UTF_8);
        ex.getResponseHeaders().set("Content-Type", "application/json; charset=UTF-8");
        // 简易 CORS，方便直接本地打开 html 调试
        ex.getResponseHeaders().set("Access-Control-Allow-Origin", "*");
        ex.sendResponseHeaders(200, body.length);
        try (OutputStream os = ex.getResponseBody()) { os.write(body); }
    }

    private static String esc(String s) {
        if (s == null) return "";
        return s.replace("\\", "\\\\").replace("\"", "\\\"")
                .replace("\n", "\\n").replace("\r", "\\r").replace("\t", "\\t");
    }

    private static void recordAddr(String address) throws IOException {
        java.util.Properties prop = new java.util.Properties();
        org.springframework.core.io.Resource res =
                new org.springframework.core.io.ClassPathResource("contract.properties");
        File f = res.getFile();
        if (f.exists()) {
            try (java.io.InputStream in = new java.io.FileInputStream(f)) { prop.load(in); }
        }
        prop.setProperty("agriTraceAddress", address);
        try (java.io.FileOutputStream out = new java.io.FileOutputStream(f)) {
            prop.store(out, "contract address");
        }
    }
}
```

> 说明：这里没用 `AgriTraceClient` 内部私有字段，而是把它的连接逻辑内联了一份（保证独立运行）。
> 若你更想复用，也可以把 `AgriTraceClient` 的方法改成 public 后直接 new 出来用，效果一样。

---

## 2. 新建 `http_run.sh`

路径：`asset-app/tool/http_run.sh`

```bash
#!/bin/bash
# 启动农产品溯源 Web 服务（HTTP 桥接）
java -Djdk.tls.namedGroups="secp256k1" \
     -Dagri.web="web" \
     -cp 'apps/*:conf/:lib/*' \
     org.fisco.bcos.asset.client.AgriTraceServer
```

`-Djdk.tls.namedGroups="secp256k1"` 别省（JDK 14 默认无该曲线，加了才能连链，和你 agri_run.sh 一样）。

> 若 `build.gradle` 的 tool 拷贝规则只认 `*_run.sh` 之外的模式，注意把 `http_run.sh` 一起带进 `dist/`。
> 多数 FISCO asset-app 模板是 `from('tool') { include '*.sh' }` 或整目录拷贝，`http_run.sh` 会自动过去。
> 若没有，手动 `cp tool/http_run.sh dist/`。

---

## 3. 放网页文件

把 `农产品溯源-前端界面.html` 复制成 `dist/web/index.html`：

```bash
mkdir -p ~/fisco/asset-app/dist/web
cp ~/fisco/asset-app/农产品溯源-前端界面.html ~/fisco/asset-app/dist/web/index.html
```

（每次 `gradle clean build` 会重建 dist，届时重新拷一次即可。也可把 web 目录放进 `src/main/resources/web`，让构建自动打包。）

---

## 4. 构建并启动

```bash
cd ~/fisco/asset-app
chmod +x tool/http_run.sh
~/opt/gradle-6.3/bin/gradle clean build      # 记住：用本地 gradle，不要用 ./gradlew

cd dist
cp ../农产品溯源-前端界面.html web/index.html 2>/dev/null || (mkdir -p web && cp ../农产品溯源-前端界面.html web/index.html)
bash http_run.sh
```

看到：

```
SDK 初始化完成, 账户地址 = 0x....
========================================
 农产品溯源系统已启动
 浏览器打开: http://localhost:8080
========================================
```

浏览器打开 `http://localhost:8080` → 依次 ① 部署 → ② 登记 → ③ 添加 → ④ 查询。

---

## 5. 常见问题

| 现象 | 原因 / 处理 |
|------|------------|
| 页面能开但点按钮"无法连接后端" | 没启动 `http_run.sh`，或端口被占（改 PORT） |
| 部署成功但查询报"尚未部署" | 已部署但进程重启后没读到地址；重新在页面点一次部署，或确认 `conf/contract.properties` 有 `agriTraceAddress` |
| 登记报 `batch already registered` | 同批次重复登记，换 batchId |
| 添加报 `batch not registered` | 先登记产品再添加溯源 |
| 连链失败 / TLS 报错 | 确认 `-Djdk.tls.namedGroups="secp256k1"` 没漏 |
| 中文乱码 | 前端 fetch 用 encodeURIComponent，后端用 URLDecoder("UTF-8")，已处理；若仍乱检查 `-Dfile.encoding=utf-8` |

---

## 6. 架构小结

该桥接层刻意用 JDK 自带 HttpServer，不引 Spring / 不下载依赖，规避你环境里的外网不稳定问题。
真实生产可换成 Spring Boot + REST，但接口形状（/api/* 返回 `{success,data,msg}`）不用变，前端无需改动。
