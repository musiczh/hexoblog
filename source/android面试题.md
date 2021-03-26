## 计算机网络

1. TCP可靠传输的原理

   > TCP使用

2. 地址栏输入url发生了什么？经过哪些协议

   > 1. 对url进行解析，利用DNS系统查找目标域名IP地址
   > 2. 发起TCP连接请求，经过三次握手与服务器建立TCP连接
   > 3. 建立HTTPS的ssl安全连接
   > 4. 通过安全连接向服务器发送HTTP请求
   > 5. 服务器响应HTTP请求
   > 6. 客户端拿到响应报文进行解析拿到页面数据，经过浏览器渲染为页面

3. HTTP与HTTPS的区别是什么？

   > ff

4. HTTPS使用了那些加密算法来解决安全问题？

   > ff

5. 



## Android

1. 如何实现布局优化？使用过什么工具实现布局优化？

   > 1. 减少布局层级；布局层级越多，view的绘制需要进行更多的递归计算，复杂度呈指数级别上涨。
   > 2. 使用更加简单的布局容器； FrameLayout与LinearLayout的性能要比constraintLayout和RelaxtiveLayout的性能更佳4。
   > 3. 减少无用的控件；
   > 4. include、merge、viewStub标签
   >
   > 布局优化：https://developer.android.google.cn/training/improving-layouts
   >
   > 布局优化辅助工具：https://developer.android.google.cn/studio/debug/layout-inspector

2. 如何解决滑动嵌套问题？

   > f

3. 说说自定义view的流程

   > f

4. 你是如何进行多线程通信的？说说Handler的原理

   > f

5. 遇到过什么内存泄露的场景？handler内存泄露的原因以及如何解决？

   >

6. RecyclerView滑动卡顿的原因与优化

   >

7. Android界面卡顿的根本原因

   >卡帧

8. 如何计算键盘的高度

   > f

9. 原生开发与跨平台flutter的区别

   > d

10. 如何实现应用内自动更新？

    > dfd

11. Android中有哪些IPC通信机制？各自的优缺点是什么？

    > ff

12. Binder的内部结构，Server是如何与ServerManager通信的？ServerManager如何被创建出来（先有鸡还是先有蛋）

    > dd

13. 聊聊三大架构模式MVP、MVC、MVVM，以及各自的优缺点

    > f

14. 如何实现进程保活与服务保活？

    > dd

15. 夜间模式的实现

    > dd



## 开源框架

1. AsyncTask的原理是什么？

   >ff

2. 为什么选择使用Glide来加载图片？Glide的缓存原理是什么？

   > f

3. Meterial Design与传统的设计有什么不同？

   >传统的设计有两种：扁平化与拟物化；
   >
   >拟物化主张把控件设计为与现实相同的模型，立体、3D是他的标签；但过于拟物化会使得界面繁杂，出现视觉疲劳；
   >
   >扁平化反其道走之，主张回归最朴素的设计，注重与功能本身，减少各种立体、阴影等等的设计，只有颜色、文字；
   >
   >他们的代表系统有锤子、魅族。
   >
   >而MD则合并了这两者的特点。MD的重点在于：交互。一个平平无奇的按钮，当按下时，高度变低、形状变小、出现水波纹，反馈感更好；他的设计让用户觉得这是一个有生命的app，而不是死板的界面。设计上，MD保持了扁平化设计的简洁，同时增加了拟物化的高度、立体因素。现代系统小米、vivo都是类似的设计。

4. 讲讲Leakcanary的原理

   >

5. EeventBus的原理？他的优缺点是什么？

   > f

6. 



## Java

1. 简述一下cocurrentHashMap的原理

   >f

2. 如何解决滑动嵌套问题？

   > f

3. 聊聊四大引用

   > d

4. 垃圾收集算法有哪些？有什么优缺点？

   > f

5. okHttp的缓存机制是什么？他的内部是如何实现缓存的？

   > dd

6. 



## 算法题

1. 最长不重复子串

   > https://leetcode-cn.com/problems/zui-chang-bu-han-zhong-fu-zi-fu-de-zi-zi-fu-chuan-lcof/

2. 计算 123456789*987654321

   > https://leetcode-cn.com/problems/multiply-strings/

3. 各种排序算法的复杂度分析

4. 字符串的所有组合

5. 反转链表



## 聊生活

1. 什么时候实习/实习多久？
2. 对开源的想法
3. 有没有写博客
4. 有没有用过xx公司的产品、xx公司的产品好在哪、如何设计一个xx公司的yy产品
5. 做什么最让你有成就感
6. 未来的职业/学习规划
7. 有没有女朋友
8. 项目做了多久/做了什么优化/遇到问题怎么去解决
9. 投了什么/多少公司、拿到多少offer
10. 进入xx公司之后，最想学习什么

