#path基本操作
##直线与点操作
```
//lineTo(float x,float y)
Path path=new Path();
path.lineTo(300,200);   //默认起点为0,0也就是屏幕左上角，canvas.translate(float x,float y)可以重置原点位置
path.lineTo(100,200);  //后一个点的起点为前一个点的终点
cavans.drawPath(path,mPaint);

```
//moveTo(float x,float y)
Path path=new Path();
path.moveTo(100,200);   //将下次路径的起点移至100,200

```
//setLastPoint(float x,float y)    //更改上次路径终点的位置
Path path=new Path();
path.lineTo(100,200);
path.setLastPoint(500,500);   //将终点改为500,500

```
//close()    封闭当前路径,画出一个闭合的图形
```
//绘制贝塞尔曲线
public void quadTo(x1, y1, x2, y2)  // 二阶贝塞尔曲线，(x1,y1) 为控制点，(x2,y2)为结束点
public void cubicTo(x1, y1, x2, y2, x3, y3)  //三阶贝塞尔曲线，(x1,y1) 为控制点，(x2,y2)为控制点，(x3,y3) 为结束点
##图形操作
```ruby
//其中的Direction的意思是 方向，趋势，是一个枚举值，CW顺时针，CCW逆时针

//矩形             
public void addRect(RectF rect, Direction dir)

public void addRect(float left, float top, float right, float bottom, Direction dir)

//圆形
public void addCircle(float x, float y, float radius, Direction dir)


//圆角矩形
//radii是一个长度为8的数组，分别为四个角的rx，ry
public void addRoundRect(RectF rect, float[] radii, Direction dir)

public void addRoundRect (float left,float top,float right,float bottom,float rx,float ry,Path.Direction dir)

public void addRoundRect (RectF rect,float[] radii,Path.Direction dir)

public void addRoundRect (float left,float top,float right,float bottom,float[] radii,Path.Direction dir)

//椭圆
public void addOval(RectF oval, Direction dir)

public void addOval (float left,float top,float right,float bottom,Path.Direction dir)

//圆弧
//直接添加一段圆弧
public void addArc (RectF oval, float startAngle, float sweepAngle)
//添加一段圆弧，如果圆弧的起点与上一次Path操作的终点不一样的话，就会在这两个点连成一条直线
public void arcTo (RectF oval, float startAngle, float sweepAngle)
//param forceMoveTo true 连接，false 不连接
public void arcTo (RectF oval, float startAngle, float sweepAngle, boolean forceMoveTo)

// 添加Path，将两个path合并
public void addPath (Path src)
//params dx,dy src相对于原来path的偏移量
public void addPath (Path src, float dx, float dy)
public void addPath (Path src, Matrix matrix)
```
##设置方法
```
//将新的path赋值给当前path
public void set(Path src)
//将当前的path进行平移
//params dx,dy,相对于当前path的平移量
 public void offset (float dx, float dy)
 //将当前path进行平移并存储到dst中
 public void offset (float dx, float dy, Path dst)
 //将path的所有操作都清空掉
 public void reset()  //会保留FillType
 public void rewind()   //不会会保留FillType
 ##判断方法
 //(after api21)判断path是否为凸多边形，如果是就为true，反之为false
 public boolean isConvex ()
 //判断path是否为空
 public boolean isEmpty ()
 //判断path是否是一个矩形，如果是一个矩形的话，将矩形的信息存到参数rect中
 public boolean isRect (RectF rect)
 