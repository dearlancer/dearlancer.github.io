#path��������
##ֱ��������
```
//lineTo(float x,float y)
Path path=new Path();
path.lineTo(300,200);   //Ĭ�����Ϊ0,0Ҳ������Ļ���Ͻǣ�canvas.translate(float x,float y)��������ԭ��λ��
path.lineTo(100,200);  //��һ��������Ϊǰһ������յ�
cavans.drawPath(path,mPaint);

```
//moveTo(float x,float y)
Path path=new Path();
path.moveTo(100,200);   //���´�·�����������100,200

```
//setLastPoint(float x,float y)    //�����ϴ�·���յ��λ��
Path path=new Path();
path.lineTo(100,200);
path.setLastPoint(500,500);   //���յ��Ϊ500,500

```
//close()    ��յ�ǰ·��,����һ���պϵ�ͼ��
```
//���Ʊ���������
public void quadTo(x1, y1, x2, y2)  // ���ױ��������ߣ�(x1,y1) Ϊ���Ƶ㣬(x2,y2)Ϊ������
public void cubicTo(x1, y1, x2, y2, x3, y3)  //���ױ��������ߣ�(x1,y1) Ϊ���Ƶ㣬(x2,y2)Ϊ���Ƶ㣬(x3,y3) Ϊ������
##ͼ�β���
```ruby
//���е�Direction����˼�� �������ƣ���һ��ö��ֵ��CW˳ʱ�룬CCW��ʱ��

//����             
public void addRect(RectF rect, Direction dir)

public void addRect(float left, float top, float right, float bottom, Direction dir)

//Բ��
public void addCircle(float x, float y, float radius, Direction dir)


//Բ�Ǿ���
//radii��һ������Ϊ8�����飬�ֱ�Ϊ�ĸ��ǵ�rx��ry
public void addRoundRect(RectF rect, float[] radii, Direction dir)

public void addRoundRect (float left,float top,float right,float bottom,float rx,float ry,Path.Direction dir)

public void addRoundRect (RectF rect,float[] radii,Path.Direction dir)

public void addRoundRect (float left,float top,float right,float bottom,float[] radii,Path.Direction dir)

//��Բ
public void addOval(RectF oval, Direction dir)

public void addOval (float left,float top,float right,float bottom,Path.Direction dir)

//Բ��
//ֱ�����һ��Բ��
public void addArc (RectF oval, float startAngle, float sweepAngle)
//���һ��Բ�������Բ�����������һ��Path�������յ㲻һ���Ļ����ͻ���������������һ��ֱ��
public void arcTo (RectF oval, float startAngle, float sweepAngle)
//param forceMoveTo true ���ӣ�false ������
public void arcTo (RectF oval, float startAngle, float sweepAngle, boolean forceMoveTo)

// ���Path��������path�ϲ�
public void addPath (Path src)
//params dx,dy src�����ԭ��path��ƫ����
public void addPath (Path src, float dx, float dy)
public void addPath (Path src, Matrix matrix)
```
##���÷���
```
//���µ�path��ֵ����ǰpath
public void set(Path src)
//����ǰ��path����ƽ��
//params dx,dy,����ڵ�ǰpath��ƽ����
 public void offset (float dx, float dy)
 //����ǰpath����ƽ�Ʋ��洢��dst��
 public void offset (float dx, float dy, Path dst)
 //��path�����в�������յ�
 public void reset()  //�ᱣ��FillType
 public void rewind()   //����ᱣ��FillType
 ##�жϷ���
 //(after api21)�ж�path�Ƿ�Ϊ͹����Σ�����Ǿ�Ϊtrue����֮Ϊfalse
 public boolean isConvex ()
 //�ж�path�Ƿ�Ϊ��
 public boolean isEmpty ()
 //�ж�path�Ƿ���һ�����Σ������һ�����εĻ��������ε���Ϣ�浽����rect��
 public boolean isRect (RectF rect)
 