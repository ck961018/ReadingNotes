# TopoDS_Edge

## 一、BRep_TEdge数据结构

BRep_TEdge是实际保存拓扑边数据的结构，它包含三个成员变量：  

* Standard_Real myTolerance // 容差
* Standard_Integer myFlags // 标志位
* BRep_ListOfCurveRepresentation myCurves // 曲线链表  

标志位在不同的二进制位分别表示曲线范围是否一致、参数表达是否一致和是否退化；  
曲线链表中保存着BRep_CurveRepresentation类型，它派生了Curve3D、CurveOnSurface、Polygon3D等几何曲线类型。

## 二、BRepLib_MakeEdge

BRepLib_MakeEdge中的拓扑边构建主要由两个Init函数实现：

```cpp
void BRepLib_MakeEdge::Init(const Handle(Geom_Curve)& CC,
                            const TopoDS_Vertex&      VV1,
                            const TopoDS_Vertex&      VV2,
                            const Standard_Real       pp1,
                            const Standard_Real       pp2)

void BRepLib_MakeEdge::Init(const Handle(Geom2d_Curve)& CC,
                            const Handle(Geom_Surface)& S,
                            const TopoDS_Vertex&        VV1,
                            const TopoDS_Vertex&        VV2,
                            const Standard_Real         pp1,
                            const Standard_Real         pp2)
```

其它函数都是它们的简单封装。

下面对这两个函数进行讲解：

### 1.通过Geom_Curve直接构建

首先用两个指针``C``和``CT``来获取``CC``底层的非trimmed类型，并将该类型保存在``C``中：

```cpp
void BRepLib_MakeEdge::Init(const Handle(Geom_Curve)& CC,
                            const TopoDS_Vertex&      VV1,
                            const TopoDS_Vertex&      VV2,
                            const Standard_Real       pp1,
                            const Standard_Real       pp2)
{
  // kill trimmed curves
  Handle(Geom_Curve)        C = CC;
  Handle(Geom_TrimmedCurve) CT = Handle(Geom_TrimmedCurve)::DownCast(C);
  while (!CT.IsNull())
  {
    C = CT->BasisCurve();
    CT = Handle(Geom_TrimmedCurve)::DownCast(C);
  }
```

接着检查输入合法性，若输入点在曲线范围外或者距离过近，则抛出异常：

```cpp
  // check parameters
  Standard_Real     p1 = pp1;
  Standard_Real     p2 = pp2;
  Standard_Real     cf = C->FirstParameter();
  Standard_Real     cl = C->LastParameter();
  Standard_Real     epsilon = Precision::PConfusion();
  Standard_Boolean  periodic = C->IsPeriodic();
  GeomAdaptor_Curve aCA(C);

  TopoDS_Vertex V1, V2;
  if (periodic)
  {
    // 将p1和p2调整到单个周期内
    ElCLib::AdjustPeriodic(cf, cl, epsilon, p1, p2);
    V1 = VV1;
    V2 = VV2;
  }
  else
  {
    // 若逆序，则交换
    if (p1 < p2)
    {
      V1 = VV1;
      V2 = VV2;
    }
    else
    {
      V2 = VV1;
      V1 = VV2;
      Standard_Real x = p1;
      p1 = p2;
      p2 = x;
    }

    // check range
    if ((cf - p1 > epsilon) || (p2 - cl > epsilon))
    {
      myError = BRepLib_ParameterOutOfRange;
      return;
    }

    // check ponctuallity
    if ((p2 - p1) <= gp::Resolution())
    {
      myError = BRepLib_LineThroughIdenticPoints;
      return;
    }
  }
```

计算输入参数在曲线上对应的点，并检查是否闭合，为后面的输入一致性检查作准备：

```cpp
  // compute points on the curve
  Standard_Boolean p1inf = Precision::IsNegativeInfinite(p1);
  Standard_Boolean p2inf = Precision::IsPositiveInfinite(p2);
  gp_Pnt           P1, P2;
  if (!p1inf)
    P1 = aCA.Value(p1);
  if (!p2inf)
    P2 = aCA.Value(p2);

  Standard_Real preci = BRepLib::Precision();
  BRep_Builder  B;

  // check for closed curve
  Standard_Boolean closed = Standard_False;
  Standard_Boolean degenerated = Standard_False;
  if (!p1inf && !p2inf)
    closed = (P1.Distance(P2) <= preci);
```

针对闭合和非闭合两种情况分别检查输入是否合法及一致：

```cpp
  // check if the vertices are on the curve
  if (closed)
  {
    if (V1.IsNull() && V2.IsNull())
    {
      B.MakeVertex(V1, P1, preci);
      V2 = V1;
    }
    else if (V1.IsNull())
      V1 = V2;
    else if (V2.IsNull())
      V2 = V1;
    else
    {
      // 如果闭合但输入点不重合，则抛出异常
      if (!V1.IsSame(V2))
      {
        myError = BRepLib_DifferentPointsOnClosedCurve;
        return;
      }
      // 如果输入参数对应曲线上的点与输入点之间的距离大于输入点的容差，则抛出异常
      else if (P1.Distance(BRep_Tool::Pnt(V1)) > Max(preci, BRep_Tool::Tolerance(V1)))
      {
        myError = BRepLib_DifferentPointsOnClosedCurve;
        return;
      }
      // 如果闭合曲线中点与边界点距离过近，则判断为退化
      else
      {
        gp_Pnt PM = aCA.Value((p1 + p2) / 2);
        if (P1.Distance(PM) < preci)
          degenerated = Standard_True;
      }
    }
  }
  else
  { // not closed
    if (p1inf)
    {
      // 如果输入参数无限大或无限小，但输入点不为空，则输入不一致
      if (!V1.IsNull())
      {
        myError = BRepLib_PointWithInfiniteParameter;
        return;
      }
    }
    else
    {
      // 输入点可以为空
      if (V1.IsNull())
      {
        B.MakeVertex(V1, P1, preci);
      }
      else if (P1.Distance(BRep_Tool::Pnt(V1)) > Max(preci, BRep_Tool::Tolerance(V1)))
      {
        myError = BRepLib_DifferentsPointAndParameter;
        return;
      }
    }

    if (p2inf)
    {
      if (!V2.IsNull())
      {
        myError = BRepLib_PointWithInfiniteParameter;
        return;
      }
    }
    else
    {
      if (V2.IsNull())
      {
        B.MakeVertex(V2, P2, preci);
      }
      else if (P2.Distance(BRep_Tool::Pnt(V2)) > Max(preci, BRep_Tool::Tolerance(V2)))
      {
        myError = BRepLib_DifferentsPointAndParameter;
        return;
      }
    }
  }
```

最后，构建一个新的拓扑边结构，添加顶点、设置范围并标注是否退化：

```cpp
  // 设置顶点方向
  V1.Orientation(TopAbs_FORWARD);
  V2.Orientation(TopAbs_REVERSED);
  myVertex1 = V1;
  myVertex2 = V2;

  TopoDS_Edge& E = TopoDS::Edge(myShape);
  // 创建拓扑边
  B.MakeEdge(E, C, preci);
  if (!V1.IsNull())
  {
    // 添加顶点
    B.Add(E, V1);
  }
  if (!V2.IsNull())
  {
    B.Add(E, V2);
  }
  // 设置范围
  B.Range(E, p1, p2);
  // 标注是否退化
  B.Degenerated(E, degenerated);

  myError = BRepLib_EdgeDone;
  Done();
```

### 2.基于Geom2d_Curve和Geom_Surface进行构建

通过曲面来计算点，其它部分与前一个Init函数基本一致：

```cpp
void BRepLib_MakeEdge::Init(const Handle(Geom2d_Curve)& CC,
                            const Handle(Geom_Surface)& S,
                            const TopoDS_Vertex&        VV1,
                            const TopoDS_Vertex&        VV2,
                            const Standard_Real         pp1,
                            const Standard_Real         pp2)
{
  // kill trimmed curves
  Handle(Geom2d_Curve)        C = CC;
  Handle(Geom2d_TrimmedCurve) CT = Handle(Geom2d_TrimmedCurve)::DownCast(C);
  while (!CT.IsNull())
  {
    C = CT->BasisCurve();
    CT = Handle(Geom2d_TrimmedCurve)::DownCast(C);
  }

  // check parameters
  Standard_Real    p1 = pp1;
  Standard_Real    p2 = pp2;
  Standard_Real    cf = C->FirstParameter();
  Standard_Real    cl = C->LastParameter();
  Standard_Real    epsilon = Precision::PConfusion();
  Standard_Boolean periodic = C->IsPeriodic();

  TopoDS_Vertex    V1, V2;
  Standard_Boolean reverse = Standard_False;

  if (periodic)
  {
    // adjust in period
    ElCLib::AdjustPeriodic(cf, cl, epsilon, p1, p2);
    V1 = VV1;
    V2 = VV2;
  }
  else
  {
    // reordonate
    if (p1 < p2)
    {
      V1 = VV1;
      V2 = VV2;
    }
    else
    {
      V2 = VV1;
      V1 = VV2;
      Standard_Real x = p1;
      p1 = p2;
      p2 = x;
      reverse = Standard_True;
    }

    // check range
    if ((cf - p1 > epsilon) || (p2 - cl > epsilon))
    {
      myError = BRepLib_ParameterOutOfRange;
      return;
    }
  }

  // compute points on the curve
  Standard_Boolean p1inf = Precision::IsNegativeInfinite(p1);
  Standard_Boolean p2inf = Precision::IsPositiveInfinite(p2);
  gp_Pnt           P1, P2;
  gp_Pnt2d         P2d1, P2d2;
  if (!p1inf)
  {
    P2d1 = C->Value(p1);
    P1 = S->Value(P2d1.X(), P2d1.Y());
  }
  if (!p2inf)
  {
    P2d2 = C->Value(p2);
    P2 = S->Value(P2d2.X(), P2d2.Y());
  }

  Standard_Real preci = BRepLib::Precision();
  BRep_Builder  B;

  // check for closed curve
  Standard_Boolean closed = Standard_False;
  if (!p1inf && !p2inf)
    closed = (P1.Distance(P2) <= preci);

  // check if the vertices are on the curve
  if (closed)
  {
    if (V1.IsNull() && V2.IsNull())
    {
      B.MakeVertex(V1, P1, preci);
      V2 = V1;
    }
    else if (V1.IsNull())
      V1 = V2;
    else if (V2.IsNull())
      V2 = V1;
    else
    {
      if (!V1.IsSame(V2))
      {
        myError = BRepLib_DifferentPointsOnClosedCurve;
        return;
      }
      else if (P1.Distance(BRep_Tool::Pnt(V1)) > Max(preci, BRep_Tool::Tolerance(V1)))
      {
        myError = BRepLib_DifferentPointsOnClosedCurve;
        return;
      }
    }
  }

  else
  { // not closed

    if (p1inf)
    {
      if (!V1.IsNull())
      {
        myError = BRepLib_PointWithInfiniteParameter;
        return;
      }
    }
    else
    {
      if (V1.IsNull())
      {
        B.MakeVertex(V1, P1, preci);
      }
      else if (P1.Distance(BRep_Tool::Pnt(V1)) > Max(preci, BRep_Tool::Tolerance(V1)))
      {
        myError = BRepLib_DifferentsPointAndParameter;
        return;
      }
    }

    if (p2inf)
    {
      if (!V2.IsNull())
      {
        myError = BRepLib_PointWithInfiniteParameter;
        return;
      }
    }
    else
    {
      if (V2.IsNull())
      {
        B.MakeVertex(V2, P2, preci);
      }
      else if (P2.Distance(BRep_Tool::Pnt(V2)) > Max(preci, BRep_Tool::Tolerance(V2)))
      {
        myError = BRepLib_DifferentsPointAndParameter;
        return;
      }
    }
  }

  V1.Orientation(TopAbs_FORWARD);
  V2.Orientation(TopAbs_REVERSED);
  myVertex1 = V1;
  myVertex2 = V2;

  TopoDS_Edge& E = TopoDS::Edge(myShape);
  B.MakeEdge(E);
  B.UpdateEdge(E, C, S, TopLoc_Location(), preci);

  if (!V1.IsNull())
  {
    B.Add(E, V1);
  }
  if (!V2.IsNull())
  {
    B.Add(E, V2);
  }
  B.Range(E, p1, p2);

  if (reverse)
    E.Orientation(TopAbs_REVERSED);

  myError = BRepLib_EdgeDone;
  Done();
}
```

除了更改点的计算方式以外，仅有三处区别：

1. 不检查参数距离；
2. 不检查闭合曲线退化；
3. 需要更新方向(orientation)。

不检查参数距离是因为二维到三维的映射可能会让参数空间上不合法的点在三维空间中合法。

> 第一个Init函数中调用的MakeEdge重载中进行了UpdateEdge，因此不需要再次调用

## 三、BRep_Builder

BRep_Builder中和拓扑边相关的函数为MakeEdge和UpdateEdge。  

### 1.MakeEdge

MakeEdge的若干重载中只有一个函数是起实际作用的，其它MakeEdge都是用来调用它和UpdateEdge函数。

MakeEdge的实现很简单，首先创建一个空的BRep_TEdge对象：

```cpp
void BRep_Builder::MakeEdge (TopoDS_Edge& E)
{
  Handle(BRep_TEdge) TE = new BRep_TEdge();
```

然后判断输入合法性，如果输入为空或已被锁定，则抛出异常：

```cpp
  if(!E.IsNull() && E.Locked())
  {
    throw TopoDS_LockedShape("BRep_Builder::MakeEdge");
  }
```

最后调用MakeShape函数，将BRep_TEdge与传入的TopoDS_Edge关联起来：

```cpp
  MakeShape(E,TE);
}
```

### 2.UpdateEdge

UpdateEdge的重载函数中有部分函数是调用其它UpdateEdge的语法糖，这里只挑选包含实际实现的进行讲解。

#### 仅更新三维曲线

```cpp
void BRep_Builder::UpdateEdge (const TopoDS_Edge& E,
                               const Handle(Geom_Curve)& C,
                               const TopLoc_Location& L,
                               const Standard_Real Tol)
```

获取TEdge，然后判断是否被锁定，若已被锁定则抛出异常：

```cpp
{
  const Handle(BRep_TEdge)& TE = *((Handle(BRep_TEdge)*) &E.TShape());
  if(TE->Locked())
  {
    throw TopoDS_LockedShape("BRep_Builder::UpdateEdge");
  }
```

计算当前位置，并调用UpdateCurves函数更新几何曲线和位置信息：

```cpp
  const TopLoc_Location l = L.Predivided(E.Location());

  UpdateCurves(TE->ChangeCurves(),C,l);
```

更新公差与Modified标志位：

```cpp
  TE->UpdateTolerance(Tol);
  TE->Modified(Standard_True);
}
```

其中，UpdateCurves函数也很重要，需要详细介绍：

```cpp
static void UpdateCurves (BRep_ListOfCurveRepresentation& lcr,
                          const Handle(Geom_Curve)&       C,
                          const TopLoc_Location&          L)
```

获取曲线表示列表的迭代器，初始化一个BRep_GCurve的指针和两个实数：

```cpp
{
  BRep_ListIteratorOfListOfCurveRepresentation itcr(lcr);
  Handle(BRep_GCurve) GC;
  Standard_Real f = 0.,l = 0.;
```

遍历曲线表示列表，转换为BRep_GCurve并获取范围，若为三维曲线，则直接跳出循环：

```cpp
  while (itcr.More()) {
    GC = Handle(BRep_GCurve)::DownCast(itcr.Value());
    if (!GC.IsNull()) {
      GC->Range(f, l);
      if (GC->IsCurve3D()) break;

    }
    itcr.Next();
  }
```

如果迭代器中存在值，则表示迭代器中为三维曲线表示，直接更新曲线和位置：

```cpp
  if (itcr.More()) {
    itcr.Value()->Curve3D(C);
    itcr.Value()->Location(L);
  }
```

否则根据传入的曲线和位置创建一个新的三维曲线表示，更新范围信息，添加到列表中：

```cpp
  else {
    Handle(BRep_Curve3D) C3d = new BRep_Curve3D(C,L);
    // test if there is already a range
    if (!GC.IsNull()) {
      C3d->SetRange(f,l);
    }
    lcr.Append(C3d);
  }
```
