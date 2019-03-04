# 记录一些奇怪的问题

## spring boot jpa 
### hibernate 一个保存操作 提交的时候 show sql  打印一个insert 和一个update,其实并没有做update操作
原因是
```
在一个事物里,先执行的insert  但是实体bean 在事物提交之前设置了其他的值,执行事物提交的时候会对比这个对象是否改变
如果改变则认为是脏数据则会发起更新
看源码:
DefaultFlushEntityEventListener
onFlushEntity 处理脏数据判断执行的方法
isUpdateNecessary 判断是否有脏数据
scheduleUpdate 如果有脏数据执行更新(其实是添加了一个更新的操作,在最后一起提交)
dirtyCheck 校验是否有脏数据

EntityUpdateAction
更新事件,处理更新任务
```

### 用springboot jpa kingshard 事物提交有问题
```
前提: 
  用kingshard 做的读写分离
  springboot 版本 1.5.9
现象是: service方法指定了 @Transactional 但是最后提交的时候还是报出事物只读,猜测原因是 在service之前有读数据操作,导致认定这个请求是只读的
临时解决办法是在最早的读操作之前加上@Transactional
后台看kingshard github 问题列表有这个问题 需要再链接地址后面加上 useLocalSessionState=true
参考 <https://github.com/flike/kingshard/issues/470>
```

## ffmpeg 处理mp4视频拼接
http://yonsm.net/mp4merge/

因为 ffmpeg 是支持切分 mp4 视频的，所以我就理所当然的以为 ffmpeg 是支持视频合并。直到今天同事找我问方法，才发现一直以为的方法是错误的， mp4 不支持直接 concate（丢人了。。。），赶紧补了一下能量，从网上抓来了多种实现。

注： 这里的 mp4 指的是网上最多见的 h264+aac mpeg4 容器的方式

1). ffmpeg + mpeg

这种是网上最常见的，基本思路是将 mp4 先转码为 mpeg 文件，mpeg是支持简单拼接的，然后再转回 mp4。
```
  ffmpeg -i 1.mp4 -sameq 1.mpg
  ffmpeg -i 2.mp4 -sameq 2.mpg
  cat 1.mpg 2.mpg | ffmpeg -f mpeg -i - -sameq -vcodec mpeg4 output.mp4 这种方式弊端很明显，需要转码。而抛开转码本身会造成的质量损失，这个效率真心无法忍受。
2). MP4Box
```

这个是 gpac 搞的专门处理 mp4 的工具，由于它会自己内部处理连接部分的数据，所以可以简单的使用类似 concate 的语法：
```
  MP4Box -cat 1.mp4 -cat 2.mp4 output.mp4 问题是，还要引入一个新的工具，而不能统一用 ffmpeg。这个也不爽。更不用说在 centos 下，你需要装一堆库，然后源码编译。有兴趣的朋友可以参考：
http://howto-heaven.blogspot.jp/2011/01/how-to-install-mp4box-on-centos.html
```

3). ffmpeg + ts 终极解决方案。这个的思路是先将 mp4 转化为同样编码形式的 ts 流，因为 ts流是可以 concate 的，先把 mp4 封装成 ts ，然后 concate ts 流， 最后再把 ts 流转化为 mp4。
```
  ffmpeg -i 1.mp4 -vcodec copy -acodec copy -vbsf h264_mp4toannexb 1.ts
  ffmpeg -i 2.mp4 -vcodec copy -acodec copy -vbsf h264_mp4toannexb 2.ts
  ffmpeg -i "concat:1.ts|2.ts" -acodec copy -vcodec copy -absf aac_adtstoasc output.mp4
转自：http://blog.eryue.me/?p=135
```

## maven 打包的坑
maven打包时候资源文件 word 格式损毁
代码中读取会显示
The document appears to be corrupted and cannot be loaded
ZIP file not opening
pom文件加上
```
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-resources-plugin</artifactId>
  <configuration>
    <nonFilteredFileExtensions>
      <nonFilteredFileExtension>doc</nonFilteredFileExtension>
      <nonFilteredFileExtension>docx</nonFilteredFileExtension>
      <nonFilteredFileExtension>xlsx</nonFilteredFileExtension>
      <nonFilteredFileExtension>xls</nonFilteredFileExtension>
      <nonFilteredFileExtension>zip</nonFilteredFileExtension>
      <nonFilteredFileExtension>cer</nonFilteredFileExtension>
      <nonFilteredFileExtension>pfx</nonFilteredFileExtension>
      <nonFilteredFileExtension>py</nonFilteredFileExtension>
      <nonFilteredFileExtension>keystore</nonFilteredFileExtension>
    </nonFilteredFileExtensions>
  </configuration>
</plugin>
```
就好了, 在使用aspose-words 发现的,以为是版本的bug结果是maven导致的

