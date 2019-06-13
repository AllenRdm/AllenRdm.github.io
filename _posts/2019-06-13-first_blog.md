---
layout: post
title: '我的第一份博客'
subtitle: 'jdk1.6中https的访问的注意项'
date: 2017-04-18
categories: 技术
tags: java https httpClient
---

我在做一个webservice接口访问时，在JDK1.6的环境像使用http的方式一样去使用https的请求，结果发现运行时会有错误，这份博客就是用于记录该问题解决的方案。

## 配置环境
首先需要找到{JDK}/jre/lib/security/java.security文件，这里的{JDK}是运行的JDK的路径。在<br>
security.provider.1=sun.security.provider.Sun
security.provider.2=sun.security.rsa.SunRsaSign
security.provider.3=com.sun.net.ssl.internal.ssl.Provider
security.provider.4=com.sun.crypto.provider.SunJCE
security.provider.5=sun.security.jgss.SunProvider
security.provider.6=com.sun.security.sasl.Provider
security.provider.7=org.jcp.xml.dsig.internal.dom.XMLDSigRI
security.provider.8=sun.security.smartcardio.SunPCSC<br>
后面增加security.provider.9=org.bouncycastle.jce.provider.BouncyCastleProvider,
然后将bcprov-ext-jdk15on-1.52.jar，和bcprov-jdk15on-1.52.jar放入{JDK}/jre/lib/ext下（我用的是这个版本）。

## 请求编码
接着就可以用httpClient这个工具类来使用https请求了。
<pre><code class="language-css">
public static String get(String url, Map<String, String> header) throws Exception {
	HttpGet httpGet = new HttpGet(url);
	HttpClient httpClient = new DefaultHttpClient(); 
	if (header != null) {
		for (Object obj : header.entrySet()) {
			Entry<String, String> entry = (Entry<String, String>) obj;
			httpGet.setHeader(entry.getKey(), entry.getValue());
		}
	}
	HttpResponse response = httpClient.execute(httpGet);
	return EntityUtils.toString(response.getEntity());
}
</code></pre>
参数header中是需要放入请求头的参数,比如一些请求需要的token等。

至此，就解决了JDK1.6请求https的问题了。
