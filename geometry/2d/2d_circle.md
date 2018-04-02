[<< 返回到上页](../index.md)

**这里将介绍圆和椭圆的博客文章**  

**圆**  
定义一个圆的数据主要包含圆心和半径：  
```
public class Circle
{
    public Vector2 mCenter;
    public float mRadius;
}
```

1. 点与圆的关系  
点在圆内判断：  
```
bool IsInCircle(Vector2 center, float r, Vector2 p)
{
    return (p - center).sqrMagnitude <= r * r;
}
```
相等的时候表示点在圆上  

2. 线与圆的关系    
直线与圆相交判断  
直线L：Vector2 p = start + t \* dir  
圆的公式为 (x - cx)^2 + (y - cy)^2 = r^2  
(x0 + t \* dx - cx)^2 + (y0 + t \* dy- cy)^2 = r^2   
得到  
(dx^2 + dy^2)t^2 + 2((x0-cx)dx + (y0-cy)dy)t + (x0-cx)^2 + (y0-cy)^2 - r^2 = 0  
求解一元二次方程t的值，有解表示相交，否则不想交   
```
bool IsLineInsectCircle(Vector2 line1, Vector2 line2, Vector2 center, float r, ref GeoInsectPointArrayInfo insect)
{
    Vector2 direction = line2 - line1;
    float a = Vector2.Dot(direction, direction);
    Vector2 c1 = line1 - center;
    float b = 2 * Vector2.Dot(direction, c1);
    float c = Vector2.Dot(c1, c1) - r * r;
    float discrm = b * b - 4 * a * c;
    if (discrm < 0)
        return false;
    discrm = Mathf.Sqrt(discrm);
    float t1 = -b + discrm;
    float t2 = -b - discrm;
    insect.mIsIntersect = true;
    if (discrm == 0)
    {
        Vector2 temp = line1 + t1 * direction;
        insect.mHitGlobalPoint.mPointArray.Add(new Vector3(temp.x, temp.y, 0.0f));
    }
    else
    {
        Vector2 temp = line1 + t1 * direction;
        insect.mHitGlobalPoint.mPointArray.Add(new Vector3(temp.x, temp.y, 0.0f));
        temp = line1 + t2 * direction;
        insect.mHitGlobalPoint.mPointArray.Add(new Vector3(temp.x, temp.y, 0.0f));
    }
    return true;
}
```
**线段和射线需要做一些限制判断，然后按照直线与圆的关系求解t，还要判断t的取值范围**

3. 圆与圆相交  
圆与圆内切和外切，相交一个点   
圆与圆不相交(外部和内部不相交)   
圆与圆相交(相交两个点)  
a. 求两圆心的方向d和中点c  
b. d旋转90度，得到垂直方向d1    
c. 以中点c和方向d1，构造成直线  
d. 直线与圆求交  


**椭圆**  

