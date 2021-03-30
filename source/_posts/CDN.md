---
title: CDN(Content Delivery Network，内容分发网络)
categories:
  - [计算机网络]
date: 2021/01/01 20:46:25
---



CDN ( Content Delivery Network，内容分发网络）

通过将网站内容（如图片、JavaScript 、CSS、网页等）分发至全网加速节点，配合精准调度系统和边缘缓存，使最终用户可以就近获取所需内容，有效解决互联网网络拥塞问题，提高终端用户访问网站的响应速度和可用性，与此同时，可大幅降低源站压力。

#### 加速原理

假设我们（client）访问的域名为www.example.com，该域名已经接入到CDN服务器，那么请求流程就是：

HTTP请求流程说明：

1. 用户在浏览器输入要访问的网站域名www.example.com，向本地DNS发起域名解析请求。
2. 本地DNS检查缓存中是否有www.example.com的IP地址记录。如果有，则直接返回给终端用户；如果没有，则向网站授权DNS查询。
3. 网站DNS服务器解析发现域名已经解析到了CNAME：www.example.com.c.cdnhwc1.com。
4. 请求被指向CDN服务。
5. CDN对域名进行智能解析，将响应速度最快的CDN节点IP地址返回给本地DNS。
6. 用户获取响应速度最快的CDN节点IP地址。
7. 浏览器在得到最佳节点的IP地址以后，向CDN节点发出访问请求。
   - 如果该IP地址对应的节点已缓存该资源，节点将数据直接返回给用户，如图中步骤7和8，请求结束。
   - 如果该IP地址对应的节点未缓存该资源，节点回源拉取资源。获取资源后，结合用户自定义配置的缓存策略，将资源缓存至节点，如图中的北京节点，并返回给用户，请求结束。配置缓存策略的操作方法，请参见[缓存配置](https://support.huaweicloud.com/usermanual-cdn/cdn_01_0116.html)。



#### CDN是否支持对动态内容进行加速？

动态内容是指不同的请求访问中获得不同数据的内容，例如：网站中的.asp、.jsp、.php、.perl、.cgi文件、API接口和动态交互请求（post、put和patch等请求）。

对于动态请求的内容，CDN必然不能缓存实时变化的内容，需要用户每次回源访问您的源站服务器。只能是择优选择源服务器返回源服务器IP。







#### 还可用来可减轻DDoS攻击

DDoS（Distributed Denial of Service，分布式拒绝服务攻击）用多台肉鸡服务器同时请求同一服务器，耗尽该服务器的资源，从而让该服务器不能为其用户提供正常服务。





ps：直接去CDN服务提供商官网肯定能看到产品介绍和工作原理

又拍云：https://help.upyun.com/knowledge-base/cdn-product/

华为云：https://support.huaweicloud.com/productdesc-cdn/cdn_01_0109.html