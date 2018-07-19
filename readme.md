#记录了一些奇怪的问题

1.spring boot jpa 
hibernate 
一个保存操作 提交的时候 show sql  打印一个insert 和一个update,其实并没有做update操作
原因是
在一个事物里,先执行的insert  但是实体bean 在事物提交之前设置了其他的值,执行事物提交的时候会对比这个对象是否改变
如果改变则认为是脏数据则会发起更新
看源码:

DefaultFlushEntityEventListener
onFlushEntity 处理脏数据判断执行的方法
isUpdateNecessary 判断是否有脏数据
scheduleUpdate 如果有脏数据执行更新(其实是添加了一个更新的操作,在最后一起提交)