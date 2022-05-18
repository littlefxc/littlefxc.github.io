---
title: RestTemplateAndOkHttp3
date: 2021-04-07 20:01:45
tags:
- springboot
- okhttp
- RestTemplate
---

# RestTemplate 集成 OKHttp3

## 核心依赖

```xml
<dependency>
    <groupId>com.squareup.okhttp3</groupId>
    <artifactId>okhttp</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot</artifactId>
</dependency>
```

## 属性配置类 OkHttpProperties

```java
@Data
@ConfigurationProperties(prefix = "okhttp")
public class OkHttpProperties {

    /**
     * 连接超时，默认 10 秒，0 表示没有超时限制
     */
    private Integer connectTimeout = 10;

    /**
     * 响应超时，默认 10 秒，0 表示没有超时限制
     */
    private Integer readTimeout = 10;

    /**
     * 写超时，默认 10 秒，0 表示没有超时限制
     */
    private Integer writeTimeout = 10;

    /**
     * 连接池中整体的空闲连接的最大数量，默认 5 个连接数
     */
    private Integer maxIdleConnections = 5;

    /**
     * 连接空闲时间最大时间，单位秒，默认 300 秒
     */
    private Long keepAliveDuration = 300L;
}
```

## 配置类 OkHttpConfig

```java
import okhttp3.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import javax.net.ssl.SSLContext;
import javax.net.ssl.SSLSocketFactory;
import javax.net.ssl.TrustManager;
import javax.net.ssl.X509TrustManager;
import java.io.IOException;
import java.security.KeyManagementException;
import java.security.NoSuchAlgorithmException;
import java.security.SecureRandom;
import java.security.cert.CertificateException;
import java.security.cert.X509Certificate;
import java.util.concurrent.TimeUnit;

/**
 * @author fengxuechao
 * @date 2021/3/19
 */
@EnableConfigurationProperties(OkHttpProperties.class)
@Configuration
public class OkHttpConfig {

    @Autowired
    private OkHttpProperties properties;

    @Bean
    public OkHttpClient okHttpClient() {
        return new OkHttpClient.Builder()
                .sslSocketFactory(sslSocketFactory(), x509TrustManager())
                // 是否开启缓存
                .retryOnConnectionFailure(false)
                .connectionPool(pool())
                .connectTimeout(properties.getConnectTimeout(), TimeUnit.SECONDS)
                .readTimeout(properties.getReadTimeout(), TimeUnit.SECONDS)
                .writeTimeout(properties.getWriteTimeout(), TimeUnit.SECONDS)
                .hostnameVerifier((hostname, session) -> true)
                // 设置代理
//            	.proxy(new Proxy(Proxy.Type.HTTP, new InetSocketAddress("127.0.0.1", 8888)))
                // 拦截器
//                .addInterceptor()
                .addNetworkInterceptor(new Interceptor() {
                    @Override
                    public Response intercept(Chain chain) throws IOException {
                        Request request = chain.request().newBuilder().addHeader("Connection", "close").build();
                        return chain.proceed(request);
                    }
                })
                .build();
    }

    @Bean
    public X509TrustManager x509TrustManager() {
        return new X509TrustManager() {
            @Override
            public void checkClientTrusted(X509Certificate[] chain, String authType)
                    throws CertificateException {
            }

            @Override
            public void checkServerTrusted(X509Certificate[] chain, String authType)
                    throws CertificateException {
            }

            @Override
            public X509Certificate[] getAcceptedIssuers() {
                return new X509Certificate[0];
            }
        };
    }

    @Bean
    public SSLSocketFactory sslSocketFactory() {
        try {
            // 信任任何链接
            SSLContext sslContext = SSLContext.getInstance("TLS");
            sslContext.init(null, new TrustManager[]{x509TrustManager()}, new SecureRandom());
            return sslContext.getSocketFactory();
        } catch (NoSuchAlgorithmException | KeyManagementException e) {
            e.printStackTrace();
        }
        return null;
    }

    @Bean
    public ConnectionPool pool() {
        return new ConnectionPool(
                properties.getMaxIdleConnections(),
                properties.getKeepAliveDuration(),
                TimeUnit.SECONDS);
    }
}
```

## OKHttp3 工具类

```java
/**
 * @author fengxuechao
 * @date 2021/3/19
 */
@Slf4j
@Component
public final class OkHttpUtil {
    private static final MediaType JSON = MediaType.parse("application/json; charset=utf-8");
    private static final MediaType XML = MediaType.parse("application/xml; charset=utf-8");

    @Autowired
    private OkHttpClient okHttpClient;

    /**
     * get 请求
     * @param url       请求url地址
     * @return string
     * */
    public String doGet(String url) {
        return doGet(url, null, null);
    }


    /**
     * get 请求
     * @param url       请求url地址
     * @param params    请求参数 map
     * @return string
     * */
    public String doGet(String url, Map<String, String> params) {
        return doGet(url, params, null);
    }

    /**
     * get 请求
     * @param url       请求url地址
     * @param headers   请求头字段 {k1, v1 k2, v2, ...}
     * @return string
     * */
    public String doGet(String url, String[] headers) {
        return doGet(url, null, headers);
    }


    /**
     * get 请求
     * @param url       请求url地址
     * @param params    请求参数 map
     * @param headers   请求头字段 {k1, v1 k2, v2, ...}
     * @return string
     * */
    public String doGet(String url, Map<String, String> params, String[] headers) {
        StringBuilder sb = new StringBuilder(url);
        if (params != null && params.keySet().size() > 0) {
            boolean firstFlag = true;
            for (String key : params.keySet()) {
                if (firstFlag) {
                    sb.append("?").append(key).append("=").append(params.get(key));
                    firstFlag = false;
                } else {
                    sb.append("&").append(key).append("=").append(params.get(key));
                }
            }
        }

        Request.Builder builder = new Request.Builder();
        if (headers != null && headers.length > 0) {
            if (headers.length % 2 == 0) {
                for (int i = 0; i < headers.length; i = i + 2) {
                    builder.addHeader(headers[i], headers[i + 1]);
                }
            } else {
                log.warn("headers's length[{}] is error.", headers.length);
            }

        }

        Request request = builder.url(sb.toString()).build();
        log.info("do get request and url[{}]", sb.toString());
        return execute(request);
    }

    /**
     * post 请求
     * @param url       请求url地址
     * @param params    请求参数 map
     * @return string
     */
    public String doPost(String url, Map<String, String> params) {
        FormBody.Builder builder = new FormBody.Builder();

        if (params != null && params.keySet().size() > 0) {
            for (String key : params.keySet()) {
                builder.add(key, params.get(key));
            }
        }
        Request request = new Request.Builder().url(url).post(builder.build()).build();
        log.info("do post request and url[{}]", url);

        return execute(request);
    }


    /**
     * post 请求, 请求数据为 json 的字符串
     * @param url       请求url地址
     * @param json      请求数据, json 字符串
     * @return string
     */
    public String doPostJson(String url, String json) {
        log.info("do post request and url[{}]", url);
        return exectePost(url, json, JSON);
    }

    /**
     * post 请求, 请求数据为 xml 的字符串
     * @param url       请求url地址
     * @param xml       请求数据, xml 字符串
     * @return string
     */
    public String doPostXml(String url, String xml) {
        log.info("do post request and url[{}]", url);
        return exectePost(url, xml, XML);
    }


    private String exectePost(String url, String data, MediaType contentType) {
        RequestBody requestBody = RequestBody.create(contentType, data);
        Request request = new Request.Builder().url(url).post(requestBody).build();

        return execute(request);
    }

    private String execute(Request request) {
        Response response = null;
        try {
            response = okHttpClient.newCall(request).execute();
            if (response.isSuccessful()) {
                return response.body().string();
            }
        } catch (Exception e) {
            log.error("OkHttp 调用失败", e);
        } finally {
            if (response != null) {
                response.close();
            }
        }
        return "";
    }
}
```

