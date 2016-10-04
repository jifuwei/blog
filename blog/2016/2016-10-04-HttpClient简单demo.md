## httpclient简单demo(入门篇)

### 使用HttpClient发送请求、接收响应很简单，一般需要如下几步即可：

- 创建CloseableHttpClient对象
- 创建请求方法的实例，并指定请求URL。如果需要发送GET请求，创建HttpGet对象；如果需要发送POST请求，创建HttpPost对象
- 如果需要发送请求参数，可可调用setEntity(HttpEntity entity)方法来设置请求参数。setParams方法已过时（4.4.1版本）。
- 调用HttpGet、HttpPost对象的setHeader(String name, String value)方法设置header信息，或者调用setHeaders(Header[] headers)设置一组header信息。
- 调用CloseableHttpClient对象的execute(HttpUriRequest request)发送请求，该方法返回一个CloseableHttpResponse。
- 调用HttpResponse的getEntity()方法可获取HttpEntity对象，该对象包装了服务器的响应内容。程序可通过该对象获取服务器的响应内容；调用CloseableHttpResponse的getAllHeaders()、getHeaders(String name)等方法可获取服务器的响应头。
- 释放连接。无论执行方法是否成功，都必须释放连接

### 具体代码如下（httpclient-4.4.1）:

```java
/**
 * 简单httpclient实例
 *
 * @author arron
 * @date 2015年11月11日 下午6:36:49
 * @version 1.0
 */
public class SimpleHttpClientDemo {

	/**
	 * 模拟请求
	 *
	 * @param url		资源地址
	 * @param map	参数列表
	 * @param encoding	编码
	 * @return
	 * @throws ParseException
	 * @throws IOException
	 */
	public static String send(String url, Map<String,String> map,String encoding) throws ParseException, IOException{
		String body = "";

		//创建httpclient对象
		CloseableHttpClient client = HttpClients.createDefault();
		//创建post方式请求对象
		HttpPost httpPost = new HttpPost(url);

		//装填参数
		List<NameValuePair> nvps = new ArrayList<NameValuePair>();
		if(map!=null){
			for (Entry<String, String> entry : map.entrySet()) {
				nvps.add(new BasicNameValuePair(entry.getKey(), entry.getValue()));
			}
		}
		//设置参数到请求对象中
		httpPost.setEntity(new UrlEncodedFormEntity(nvps, encoding));

		System.out.println("请求地址："+url);
		System.out.println("请求参数："+nvps.toString());

		//设置header信息
		//指定报文头【Content-type】、【User-Agent】
		httpPost.setHeader("Content-type", "application/x-www-form-urlencoded");
		httpPost.setHeader("User-Agent", "Mozilla/4.0 (compatible; MSIE 5.0; Windows NT; DigExt)");

		//执行请求操作，并拿到结果（同步阻塞）
		CloseableHttpResponse response = client.execute(httpPost);
		//获取结果实体
		HttpEntity entity = response.getEntity();
		if (entity != null) {
			//按指定编码转换结果实体为String类型
			body = EntityUtils.toString(entity, encoding);
		}
		EntityUtils.consume(entity);
		//释放链接
		response.close();
        return body;
	}

	public static void main(String[] args) throws ParseException, IOException {
		String url="http://php.weather.sina.com.cn/iframe/index/w_cl.php";
		Map<String, String> map = new HashMap<String, String>();
		map.put("code", "js");
		map.put("day", "0");
		map.put("city", "上海");
		map.put("dfc", "1");
		map.put("charset", "utf-8");
		String body = send(url, map,"utf-8");
      System.out.println("交易响应结果：");
      System.out.println(body);
	}
}
```