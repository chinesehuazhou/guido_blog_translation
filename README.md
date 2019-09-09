# Guido 解析器系列文章翻译
英文目录：[https://medium.com/@gvanrossum_83706/peg-parsing-series-de5d41b2ed60](https://medium.com/@gvanrossum_83706/peg-parsing-series-de5d41b2ed60)



## 翻译进度：

1、[系列之一：PEG解析器 ](https://github.com/chinesehuazhou/guido_blog_translation/blob/master/%E8%A7%A3%E6%9E%90%E5%99%A8%E7%B3%BB%E5%88%97%E4%B9%8B%E4%B8%80%EF%BC%9APEG%20%E8%A7%A3%E6%9E%90%E5%99%A8.md)

2、[系列之二：构建一个PEG解析器 ](https://github.com/chinesehuazhou/guido_blog_translation/blob/master/%E8%A7%A3%E6%9E%90%E5%99%A8%E7%B3%BB%E5%88%97%E4%B9%8B%E4%BA%8C%EF%BC%9A%E6%9E%84%E5%BB%BA%E4%B8%80%E4%B8%AA%20PEG%20%E8%A7%A3%E6%9E%90%E5%99%A8.md)

3、[系列之三：生成一个 PEG 解析器 ](https://github.com/chinesehuazhou/guido_blog_translation/blob/master/%E8%A7%A3%E6%9E%90%E5%99%A8%E7%B3%BB%E5%88%97%E4%B9%8B%E4%B8%89%EF%BC%9A%E7%94%9F%E6%88%90%E4%B8%80%E4%B8%AA%20PEG%20%E8%A7%A3%E6%9E%90%E5%99%A8.md)

4、[系列之四：可视化 PEG 解析 ](https://github.com/chinesehuazhou/guido_blog_translation/blob/master/%E8%A7%A3%E6%9E%90%E5%99%A8%E7%B3%BB%E5%88%97%E4%B9%8B%E5%9B%9B%EF%BC%9A%E5%8F%AF%E8%A7%86%E5%8C%96%20PEG%20%E8%A7%A3%E6%9E%90.md)

5、[系列之五：左递归 PEG 语法 ](https://github.com/chinesehuazhou/guido_blog_translation/blob/master/%E8%A7%A3%E6%9E%90%E5%99%A8%E7%B3%BB%E5%88%97%E4%B9%8B%E4%BA%94%EF%BC%9A%E5%B7%A6%E9%80%92%E5%BD%92%20PEG%20%E8%AF%AD%E6%B3%95.md)

6、[解析器系列之六：给 PEG 语法添加动作](https://github.com/chinesehuazhou/guido_blog_translation/blob/master/%E8%A7%A3%E6%9E%90%E5%99%A8%E7%B3%BB%E5%88%97%E4%B9%8B%E5%85%AD%EF%BC%9A%E7%BB%99%20PEG%20%E8%AF%AD%E6%B3%95%E6%B7%BB%E5%8A%A0%E5%8A%A8%E4%BD%9C.md)





## 番外篇：

1、[Python 之父撰文回忆：为什么要创造 pgen 解析器？ ](https://github.com/chinesehuazhou/guido_blog_translation/blob/master/Python%20%E4%B9%8B%E7%88%B6%E6%92%B0%E6%96%87%E5%9B%9E%E5%BF%86%EF%BC%9A%E4%B8%BA%E4%BB%80%E4%B9%88%E8%A6%81%E5%88%9B%E9%80%A0%20pgen%20%E8%A7%A3%E6%9E%90%E5%99%A8%EF%BC%9F.md)

2、[从 Python 之父的对话聊起，关于知识产权、知识共享与文章翻译 ](https://github.com/chinesehuazhou/guido_blog_translation/blob/master/%E4%BB%8E%20Python%20%E4%B9%8B%E7%88%B6%E7%9A%84%E5%AF%B9%E8%AF%9D%E8%81%8A%E8%B5%B7%EF%BC%8C%E5%85%B3%E4%BA%8E%E7%9F%A5%E8%AF%86%E4%BA%A7%E6%9D%83%E3%80%81%E7%9F%A5%E8%AF%86%E5%85%B1%E4%BA%AB%E4%B8%8E%E6%96%87%E7%AB%A0%E7%BF%BB%E8%AF%91.md)



已建微信交流群，加豌豆花下猫（id：chinesehuazhou），邀请入群