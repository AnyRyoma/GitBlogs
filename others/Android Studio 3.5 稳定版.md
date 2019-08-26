##  Android Studio 3.5 稳定版

##### 采坑记录

1.xml布局文件用快捷键进行代码格式化时,子view布局会换位置(如根布局是LinearLayout或FrameLayout时) 导致布局预览产生变化
修复方法: 

第一步：Settings -> Editor -> Code Style -> XML 
第二部：打开面板后，在面板上部，右侧有一个设置小图标，点击后选择Restore Defaults， 之后 Apply
![](https://img.lruihao.cn/imgs/2019/08/0eb277c772c9b53e.png)
第三步：重启AS即可