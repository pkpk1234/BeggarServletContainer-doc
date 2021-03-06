# HTTP请求parse流程、RequestLineParser、HttpQueryParameterParser

根据标准， HTTP请求消息格式如下：

```
Request-Line              
*(( general-header        
   | request-header         
   | entity-header ) CRLF) 
CRLF
[ message-body ]
```

所有如果要将HTTP请求从输入流中parse为Java对象，需要完成3个步骤：

1. 解析Request-Line
2. 解析headers
3. 解析body

让我们首先分析最简单的解析Request-Line。

## 解析Request-Line

根据标准Request-Line的格式如下：

```
Method SP Request-URI SP HTTP-Version CRLF
```

其中Request-URI是客户端使用URLEncode之后的URI，即如果元素URI中包含非ASCII字符，客户端必须将其编码为 %编码值 的形式。

分为4段：

1. 第一段是HTTP方法名，即GET、POST、PUT等。
2. 第二段是请求路径，如 /index.html
3. 第三段是HTTP协议版本，一般是HTTP/1.1。
4. 最后是一个CRLF回车换行。

除了CRLF，段与段之间用空格分隔。综上所述，解析步骤为：

1. 去掉行尾的CRLF，并trim两端的空格。
2. 使用空格将字符串分为大小为3的数组a。
3. method是a\[0\]，Request-URI是a\[1\]，HTTP-Version是a\[2\]。

代码如下：

```java
public class DefaultRequestLineParser implements RequestLineParser {
    private static final String SPLITTER = "\\s+";
    private static final String CRLF = "\r\n";

    @Override
    public RequestLine parse(String startLine) {
        //去掉末尾的CRLF和空格，转化为Method SP Request-URI SP HTTP-Version
        String str = startLine.replaceAll(CRLF, "").trim();
        String[] parts = str.split(SPLITTER);
        //数组格式{Method,Request-URI,HTTP-Version}
        if (parts.length == 3) {
            String method = parts[0];
            URI uri = URI.create(parts[1]);
            String httpVersion = parts[2];
            return new RequestLine(method, uri, httpVersion);
        }
        throw new ParserException("startline format illegal");
    }
}
```

## 构造QueryParameters

当提交的请求包含查询信息，如 /person?age=20&gender=male，Request-URI将包含这些信息。我们可以将查询信息构造为对象，并保存到HttpMessage接口中。

解析步骤也很简单：

1. 调用Request-Line构造而来的URI的getQuery\(\)方法，获取到query字符串。
2. 使用&将query字符串拆分为key-value字符串组成的数组。
3. 遍历这个数组，将key-value字符串转换为HttpQueryParameter对象。

```java
public class DefaultHttpQueryParameterParser implements HttpQueryParameterParser {
    private final HttpQueryParameters httpQueryParameters;
    private static final String SPLITTER = "&";
    private static final String KV_SPLITTER = "=";

    public DefaultHttpQueryParameterParser() {
        this.httpQueryParameters = new HttpQueryParameters();
    }

    @Override
    public HttpQueryParameters parse(String queryString) {
        if (queryString.contains(SPLITTER)) {
            String[] keyValues = queryString.split(SPLITTER);
            for (String keyValue : keyValues) {
                if (keyValue.contains(KV_SPLITTER)) {
                    String[] temp = keyValue.split(KV_SPLITTER);
                    if (temp.length == 2) {
                        this.httpQueryParameters
                                .addQueryParameter(new HttpQueryParameter(temp[0], temp[1]));
                    }

                }
            }
        }
        return this.httpQueryParameters;
    }
}
```

```java
public class HttpQueryParameters {

    private final LinkedListMultimap<String, HttpQueryParameter> prametersMultiMap;

    public HttpQueryParameters() {
        this.prametersMultiMap = LinkedListMultimap.create();
    }


    public List<HttpQueryParameter> getQueryParameter(String name) {
        return this.prametersMultiMap.get(name);
    }

    public HttpQueryParameter getFirstQueryParameter(String name) {
        if (this.prametersMultiMap.containsKey(name)) {
            return this.prametersMultiMap.get(name).get(0);
        }
        return null;
    }

    public List<HttpQueryParameter> getQueryParameters() {
        return this.prametersMultiMap.values();
    }

    public void addQueryParameter(HttpQueryParameter httpQueryParameter) {
        this.prametersMultiMap.put(httpQueryParameter.getName(), httpQueryParameter);
    }

    public void removeQueryParameter(HttpQueryParameter httpQueryParameter) {
        this.prametersMultiMap.remove(httpQueryParameter.getName(), httpQueryParameter);
    }

    public void removeQueryParameter(String httpQueryParameter) {
        this.prametersMultiMap.removeAll(httpQueryParameter);
    }

    public boolean hasRequestParameter(String httpQueryParameter) {
        return this.prametersMultiMap.containsKey(httpQueryParameter);
    }

    public Set<String> getQueryParameterNames() {
        return this.prametersMultiMap.keySet();
    }
}
```

HttpQueryParameters中使用guava的LinkedListMultimap保存HttpQueryParameters，以支持同名参数的场景，并保证遍历时访问HttpQueryParameters的顺序，和添加时一致。

单元测试：

测试Request Line的解析

```java
public class TestDefaultRequestLineParser {
    private static final Logger
            LOGGER = LoggerFactory.getLogger(TestDefaultRequestLineParser.class);

    @Test
    public void test() {
        DefaultRequestLineParser defaultRequestLineParser
                = new DefaultRequestLineParser();
        RequestLine result = defaultRequestLineParser.parse("GET /hello.txt HTTP/1.1\r\n");
        String method = result.getMethod();
        assertEquals("GET", method);
        final URI requestURI = result.getRequestURI();
        assertEquals(URI.create("/hello.txt"), requestURI);
        LOGGER.info(requestURI.getQuery());
        LOGGER.info(requestURI.getFragment());
        assertEquals("HTTP/1.1", result.getHttpVersion());
    }

    @Test
    public void testQuery() {
        DefaultRequestLineParser defaultRequestLineParser
                = new DefaultRequestLineParser();
        RequestLine result = defaultRequestLineParser.parse("GET /test?a=123&a1=1&b=456 HTTP/1.1\r\n");
        String method = result.getMethod();
        assertEquals("GET", method);
        final URI requestURI = result.getRequestURI();
        assertEquals(URI.create("/test?a=123&a1=1&b=456"), requestURI);
        LOGGER.info(requestURI.getQuery());
        assertEquals("HTTP/1.1", result.getHttpVersion());
    }
}
```

测试Query字符串解析

```java
public class TestDefaultHttpQueryParameterParser {

    @Test
    public void test() {
        String queryStr = "a=123&a1=1&b=456&a=321";
        DefaultHttpQueryParameterParser httpRequestParameterParser
                = new DefaultHttpQueryParameterParser();
        HttpQueryParameters result = httpRequestParameterParser.parse(queryStr);
        List<HttpQueryParameter> parameters = result.getQueryParameter("a");
        assertNotNull(parameters);
        assertEquals(2, parameters.size());
        assertEquals("123", parameters.get(0).getValue());
        assertEquals("321", parameters.get(1).getValue());
    }
}
```

完整代码：https://github.com/pkpk1234/BeggarServletContainer/tree/step8/

分支step8

