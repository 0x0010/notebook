---
title: 合理使用HttpClient连接池
date: 2016/5/1 22:45:19
---

HttpClient是遵循Http协议，使用Java语言开发的Http请求客户端。其API比Java原生的HttpConnection更友好，研发使用起来也更方便。这段文字记录自己如何使用httpclient-4连接池。

本文代码依赖清单如下：

* org.apache.httpcomponents:httpclient:4.5.2
* org.apache.httpcomponents:httpcore:4.4.4
* commons-logging:commons-logging:1.2
* commons-codec:common-codec:1.9

#### 前言
HttpClient发展到现在，最新的发布版本是4。

就个人经历来说，使用过两个版本（3和4）。这两个版本之间的差异挺大，任何工具的使用都是由浅入深，其中就包含程序员写出来的代码。

刚开始使用HttpClient可能会写出这样的代码
````java
CloseableHttpClient httpClient = HttpClientBuilder.create().build();
httpClient.execute(new HttpGet("http://www.google.com"));
````
如果你能写出这样的代码，那么恭喜你，你已经学会使用httpclient。

对于优秀的项目或者系统来说，负载和流量往往会变的越来越大。在大负载的压力下，服务的tps就会有很高的要求。如果存在系统间交互，这个外系统的交互环节往往会影响服务的效率。如果采用异步处理Http交互，那么场景我们另说。不管架构上怎么处理，Http的交互肯定要存在，即使是异步，异步任务的效率也不可以慢的无法接受。

回到HttpClient上边来，如果每次请求都创建一个连接，可想效率一定不会好，在网络不稳定时，情况会更糟。

#### 使用<code>PoolingHttpClientConnectionManager</code>来管理连接。

给出一个基本的实现：
````java
import org.apache.http.client.config.RequestConfig;
import org.apache.http.client.methods.CloseableHttpResponse;
import org.apache.http.client.methods.HttpGet;
import org.apache.http.client.protocol.HttpClientContext;
import org.apache.http.impl.DefaultConnectionReuseStrategy;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClientBuilder;
import org.apache.http.impl.conn.PoolingHttpClientConnectionManager;
import org.apache.http.util.EntityUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.util.concurrent.TimeUnit;

/**
 * Pooled http client
 * Created by Sam Tsai on 2016/4/25.
 */
public class PooledHttpClient {

    private static final Logger logger = LoggerFactory.getLogger(PooledHttpClient.class);

    private static HttpClientBuilder hcb;
    private static HttpClientContext httpContext = new HttpClientContext();

    static {
        httpContext.setRequestConfig(
                RequestConfig.custom()
                        .setConnectionRequestTimeout(5000)
                        .setConnectTimeout(5000)
                        .setSocketTimeout(5000)
                        .build());

        PoolingHttpClientConnectionManager poolingConnManager = new PoolingHttpClientConnectionManager();
        poolingConnManager.setMaxTotal(20);
        poolingConnManager.setDefaultMaxPerRoute(poolingConnManager.getMaxTotal());
        poolingConnManager.closeIdleConnections(20, TimeUnit.SECONDS);

        hcb = HttpClientBuilder.create()
                .setConnectionManager(poolingConnManager)
                .disableAuthCaching()
                .disableCookieManagement()
                .setConnectionReuseStrategy(new DefaultConnectionReuseStrategy())
                .evictIdleConnections(60, TimeUnit.SECONDS);
        // Optional
        hcb.setUserAgent("Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/49.0.2623.110 Safari/537.36");
    }

    public static String doGet(String url) {
        String responseStr = null;
        CloseableHttpClient client = hcb.build();
        try {
            CloseableHttpResponse chr = client.execute(new HttpGet(url), httpContext);
            responseStr = readInputStream(chr.getEntity().getContent());
            EntityUtils.consume(chr.getEntity());
        } catch (IOException ioe) {
            logger.error(String.format("Get [%s] error!", url), ioe);
        }
        return responseStr;
    }

    private static String readInputStream(InputStream inputStream) {
        String isString = null;
        try {
            byte[] buffer = new byte[512];
            int length;
            ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
            while ((length = inputStream.read(buffer)) > 0) {
                byteArrayOutputStream.write(buffer, 0, length);
            }
            isString = byteArrayOutputStream.toString("UTF-8");
        } catch (IOException ioe) {
            logger.error("Read response stream failed.", ioe);
        }
        return isString;
    }
}
````

#### 注意点：
* Response读取之后，一定要关闭response.inputStream()。否则HttpRoute的keep-alive不会生效。 EntityUtils有consume方法可以使用。<code>EntityUtils.consume(chr.getEntity());</code>
* 务必设置<code>ConnectionRequestTimeout</code>。如果消费了Resonpse的InputStream，但是没有及时关闭，会导致Route数量激增，直至用完pool的maxtotal。当Pool满了，并且没有对应的Route是alive状态，如果没有指定超时时间，程序就会卡死。
