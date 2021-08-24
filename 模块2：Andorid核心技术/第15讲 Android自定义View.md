## 引言

自定义 UI 控件有 2 种方式：

1. 继承系统提供的成熟控件（比如 LinearLayout、RelativeLayout、ImageView 等）；
2. 直接继承自系统 View 或者 ViewGroup，并自绘显示内容。

## 继承现有控件

相对而言，这是一种较简单的实现方式。因为大部分核心工作，比如关于控件大小的测量、控件位置的摆放等相关的计算，在系统中都已经实现并封装好，开发人员只要在其基础上进行一些扩展，并按照自己的意图显示相应的 UI 元素。比如以下代码：

[![继承现有控件01.png](https://z3.ax1x.com/2021/08/11/fUHQZq.png)](https://imgtu.com/i/fUHQZq)

CustomToolBar 继承自 RelativeLayout，在构造函数中通过 addView 方式分别添加了 2 个 ImageView 和 1 个 TextView。显示效果如下：

[![继承现有控件02.png](https://z3.ax1x.com/2021/08/11/fUHcSe.png)](https://imgtu.com/i/fUHcSe)

## 自定义属性

有时候我们想在 XML 布局文件中使用 CustomToolBar 时，希望能在 XML 文件中直接指定 title 的显示内容、字体颜色，leftImage 和 rightImage 的显示图片等。这就需要使用自定义属性。
自定义属性具体步骤分为以下几步：

### attrs.xml 中声明自定义属性

在 res 的 values 目录下的 attrs.xml 文件中（没有就自己新建一个），使用 标签自定义属性，如下所示：

[![自定义属性01.png](https://z3.ax1x.com/2021/08/11/fUbek6.png)](https://imgtu.com/i/fUbek6)

解释说明：

+ **标签代表定义一个自定义属性集合，一般会与自定义控件结合使用；**

+ **标签则是某一条具体的属性，name 是属性名称，format 代表属性的格式。**

### 在 XML 布局文件中使用自定义属性

[![自定义属性02 .png](https://z3.ax1x.com/2021/08/11/fUbxud.png)](https://imgtu.com/i/fUbxud)

需要先添加命名空间 xmlns:app，然后通过命名空间 app 引用自定义属性，并传入相应的图片资源和字符串内容。

### 在 CustomToolBar 中，获取自定义属性的引用值

[![自定义属性02.png](https://z3.ax1x.com/2021/08/11/fUqeDs.png)](https://imgtu.com/i/fUqeDs)

## 直接继承自 View 或者 ViewGroup性

这种方式相比第一种麻烦一些，但是更加灵活，也能实现更加复杂的 UI 界面。一般情况下使用这种实现方式需要解决以下几个问题：

1. 如何根据相应的属性将 UI 元素绘制到界面；
2. 自定义控件的大小，也就是宽和高分别设置多少；
3. 如果是 ViewGroup，如何合理安排其内部子 View 的摆放位置。

以上 3 个问题依次在如下 3 个方法中得到解决：

1. **onDraw**
2. **onMeasure**
3. **onLayout**

因此自定义 View 的重点工作其实就是复写并合理的实现这 3 个方法。注意：并不是每个自定义 View 都需要实现这 3 个方法，大多数情况下只需要实现其中 2 个甚至 1 个方法也能满足需求。

### onDraw

onDraw 方法接收一个 Canvas 类型的参数。Canvas 可以理解为一个画布，在这块画布上可以绘制各种类型的 UI 元素。

系统提供了一系列 Canvas 操作方法，如下：

[![onDraw_01.png](https://z3.ax1x.com/2021/08/11/fUqd56.png)](https://imgtu.com/i/fUqd56)

从上图中可以看出，Canvas 中每一个绘制操作都需要传入一个 Paint 对象。Paint 就相当于一个画笔，我们可以通过设置画笔的各种属性，来实现不同绘制效果：

[![onDraw_02.png](https://z3.ax1x.com/2021/08/11/fUqhPf.png)](https://imgtu.com/i/fUqhPf)

比如如下代码，定义 PieImageView 继承自 View，然后在 onDraw 方法中，分别使用 Canvas 的 drawArc 和 drawCircle 方法来绘制弧度和圆形。这两个形状组合在一起就能表示一个简易的圆形进度条控件。

[![onDraw_03.png](https://z3.ax1x.com/2021/08/11/fULiZR.png)](https://imgtu.com/i/fULiZR)

在布局文件中直接使用上述的 PieImageView，设置宽高为 300dp，并在 Activity 中设置 PieImageView 的进度为 45，如下所示：

[![onDraw_04.png](https://z3.ax1x.com/2021/08/11/fULusH.png)](https://imgtu.com/i/fULusH)

最终运行显示效果如下：

[![onDraw_05.png](https://z3.ax1x.com/2021/08/11/fULglF.png)](https://imgtu.com/i/fULglF)

如果在上面代码中的布局文件中，将 PieImageView 的宽高设置为 wrap_content（也就是自适应），重新运行则显示效果如下：

[![onDraw_06 .png](https://z3.ax1x.com/2021/08/11/fUOpff.png)](https://imgtu.com/i/fUOpff)

很显然，PieImageView 并没有正常显示。问题的主要原因就是在 PieImageView 中并没有在 onMeasure 方法中进行重新测量，并重新设置宽高。

### onMeasure

首先我们需要弄清楚，自定义 View 为什么需要重新测量。正常情况下，我们直接在 XML 布局文件中定义好 View 的宽高，然后让自定义 View 在此宽高的区域内显示即可。但是为了更好地兼容不同尺寸的屏幕，Android 系统提供了 wrap_contetn 和 match_parent 属性来规范控件的显示规则。它们分别代表**自适应大小**和**填充父视图的大小**，但是这两个属性并没有指定具体的大小，因此我们需要在 onMeasure 方法中过滤出这两种情况，真正的测量出自定义 View 应该显示的宽高大小。

所有工作都是在 onMeasure 方法中完成，方法定义如下：

[![onMeasure_01.png](https://z3.ax1x.com/2021/08/11/fUO0je.png)](https://imgtu.com/i/fUO0je)

可以看出，方法会传入 2 个参数 widthMeasureSpec 和 heightMeasureSpec。这两个参数是从父视图传递给子 View 的两个参数，看起来很像宽、高，但是它们所表示的不仅仅是宽和高，还有一个非常重要的**测量模式**。

一共有 3 种测量模式。

1. **EXACTLY：表示在 XML 布局文件中宽高使用 match_parent 或者固定大小的宽高；**
2. **AT_MOST：表示在 XML 布局文件中宽高使用 wrap_content；**
3. **UNSPECIFIED：父容器没有对当前 View 有任何限制，当前 View 可以取任意尺寸，比如 ListView 中的 item。**

具体值和测量模式都可以通过 Android SDK 中提供的 MeasureSpec.java 类获取：

[![onMeasure_02.png](https://z3.ax1x.com/2021/08/11/fUOfgS.png)](https://imgtu.com/i/fUOfgS)

为什么 1 个 int 值可以代表 2 种意义呢？ 实际上 widthMeasureSpec 和 heightMeasureSpec 都是使用二进制高 2 位表示测量模式，低 30 位表示宽高具体大小。

#### 重新回到 PieImageView

在 PieImageView 中并没有复写 onMeasure 方法，因此默认使用父类也就是 View 中的实现，View 中的 onMeasure 默认实现如下：

[![PieImageView01.png](https://z3.ax1x.com/2021/08/11/fUjoYq.png)](https://imgtu.com/i/fUjoYq)

蓝色框中的 **setMeasuredDimension** 是一个非常重要的方法，这个方法传入的值直接决定 View 的宽高，也就是说如果调用 setMeasuredDimension(100,200)，最终 View 就显示宽 100 * 高 200 的矩形范围。红色下划线标识的 getDefaultSize 返回的是默认大小，默认为父视图的剩余可用空间。

	这也是为什么 PieImageView 显示异常的原因，虽然我们在 XML 中指定的是
	 wrap_content，但是实际使用的
	宽高值却是父视图的剩余可用空间，从代码中可以看出是整个屏幕的宽高。

问题的原因找到，解决方法只要复写 onMeasure，过滤出 wrap_content 的情况，并主动调用 setMeasuredDimension 方法设置正确的宽高即可：

[![PieImageView02.png](https://z3.ax1x.com/2021/08/11/fUvd3V.png)](https://imgtu.com/i/fUvd3V)

#### ViewGroup 中的 onMeasure

如果我们自定义的控件是一个容器，onMeasure 方法会更加复杂一些。因为 ViewGroup 在测量自己的宽高之前，需要先确定其内部子 View 的所占大小，然后才能确定自己的大小。比如如下一段代码：

[![ViewGroup 中的 onMeasure01.png](https://z3.ax1x.com/2021/08/11/fUxSbQ.png)](https://imgtu.com/i/fUxSbQ)

LinearLayout 的宽高为 wrap_content 表示由子控件的大小决定，而 3 个子控件的宽度分别为300、200、100，那最终 LinearLayout 的宽度显示多少呢？ 运行结果如下：

[![ViewGroup 中的 onMeasure02.png](https://z3.ax1x.com/2021/08/11/fUxtqe.png)](https://imgtu.com/i/fUxtqe)

可以看出 LinearLayout 的最终宽度由其内部最大的子 View 宽度决定。

当我们自己定义一个 ViewGroup 的时候，也需要在 onMeasure 方法中综合考虑子 View 的宽度。比如如果要实现一个流式布局 FlowLayout，效果如下：

[![ViewGroup 中的 onMeasure03.png](https://z3.ax1x.com/2021/08/11/fUxDRP.png)](https://imgtu.com/i/fUxDRP)

在大多数 App 的搜索界面经常会使用 FlowLayout 来展示历史搜索记录或者热门搜索项。
FlowLayout 的每一行上的 item 个数不一定，当每行的 item 累计宽度超过可用总宽度，则需要重启一行摆放 item 项。因此我们需要在 onMeasure 方法中主动的分行计算出 FlowLayout 的最终高度，如下所示：

	//测量控件的宽和高
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {

        super.onMeasure(widthMeasureSpec, heightMeasureSpec);

        //获得宽高的测量模式和测量值
        int widthMode = MeasureSpec.getMode(widthMeasureSpec);
        int widthSize = MeasureSpec.getSize(widthMeasureSpec);
        int heightSize = MeasureSpec.getSize(heightMeasureSpec);
        int heightMode = MeasureSpec.getMode(heightMeasureSpec);

        //获得容器中子View的个数
        int childCount = getChildCount();
        //记录每一行View的总宽度
        int totalLineWidth = 0;
        //记录每一行最高View的高度
        int perLineMaxHeight = 0;
        //记录当前ViewGroup的总高度
        int totalHeight = 0;
        for (int i = 0; i < childCount; i++) {
            View childView = getChildAt(i);
            //对子View进行测量
            measureChild(childView, widthMeasureSpec, heightMeasureSpec);
            MarginLayoutParams lp = (MarginLayoutParams) childView.getLayoutParams();
            //获得子View的测量宽度
            int childWidth = childView.getMeasuredWidth() + lp.leftMargin + lp.rightMargin;
            //获得子View的测量高度
            int childHeight = childView.getMeasuredHeight() + lp.topMargin + lp.bottomMargin;
            if (totalLineWidth + childWidth > widthSize) {
                //统计总高度
                totalHeight += perLineMaxHeight;
                //开启新的一行
                totalLineWidth = childWidth;
                perLineMaxHeight = childHeight;
            } else {
                //记录每一行的总宽度
                totalLineWidth += childWidth;
                //比较每一行最高的View
                perLineMaxHeight = Math.max(perLineMaxHeight, childHeight);
            }

            //当该View已是最后一个View时，将该行最大高度添加到totalHeight中
            if (i == childCount - 1) {
                totalHeight += perLineMaxHeight;
            }
        }

        //如果高度的测量模式是EXACTLY，则高度用测量值，否则用计算出来的总高度（这时高度的设置为wrap_content）
        heightSize = heightMode == MeasureSpec.EXACTLY ? heightSize : totalHeight;
        setMeasuredDimension(widthSize, heightSize);
    }

上述 onMeasure 方法的主要目的有 2 个：

+ **调用 measureChild 方法递归测量子 View；**
+ **通过叠加每一行的高度，计算出最终 FlowLayout 的最终高度 totalHeight。**

### onLayout

上面的 FlowLayout 中的 onMeasure 方法只是计算出 ViewGroup 的最终显示宽高，但是并没有规定某一个子 View 应该显示在何处位置。要定义 ViewGroup 内部子 View 的显示规则，则需要复写并实现 onLayout 方法。

ViewGroup 中的 onLayout 方法声明如下：

[![onLayout_01.png](https://z3.ax1x.com/2021/08/11/fUzJWq.png)](https://imgtu.com/i/fUzJWq)

它是一个抽象方法，也就是说每一个自定义 ViewGroup 都必须主动实现如何排布子 View，具体就是遍历每一个子 View，调用 child.(l, t, r, b) 方法来为每个子 View 设置具体的布局位置。四个参数分别代表左上右下的坐标位置，一个简易的 FlowLayout 实现如下：

    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        mAllViews.clear();
        mPerLineMaxHeight.clear();
        //存放每一行的子View
        List<View> lineViews = new ArrayList<>();
        //记录每一行已存放View的总宽度
        int totalLineWidth = 0;
        //记录每一行最高View的高度
        int lineMaxHeight = 0;
        /****遍历所有View，将View添加到List<List<View>>集合中**********/
        //获得子View的总个数
        int childCount = getChildCount();
        for (int i = 0; i < childCount; i++) {
            View childView = getChildAt(i);
            MarginLayoutParams lp = (MarginLayoutParams) childView.getLayoutParams();
            int childWidth = childView.getMeasuredWidth() + lp.leftMargin + lp.rightMargin;
            int childHeight = childView.getMeasuredHeight() + lp.topMargin + lp.bottomMargin;
            if (totalLineWidth + childWidth > getWidth()) {
                mAllViews.add(lineViews);
                mPerLineMaxHeight.add(lineMaxHeight);
                //开启新的一行
                totalLineWidth = 0;
                lineMaxHeight = 0;
                lineViews = new ArrayList<>();
            }
            totalLineWidth += childWidth;
            lineViews.add(childView);
            lineMaxHeight = Math.max(lineMaxHeight, childHeight);
        }

        //单独处理最后一行
        mAllViews.add(lineViews);
        mPerLineMaxHeight.add(lineMaxHeight);
        /************遍历集合中的所有View并显示出来************/
        //表示一个View和父容器左边的距离
        int mLeft = 0;
        //表示View和父容器顶部的距离
        int mTop = 0;
        for (int i = 0; i < mAllViews.size(); i++) {
            //获得每一行的所有View
            lineViews = mAllViews.get(i);
            lineMaxHeight = mPerLineMaxHeight.get(i);
            for (int j = 0; j < lineViews.size(); j++) {
                View childView = lineViews.get(j);
                MarginLayoutParams lp = (MarginLayoutParams) childView.getLayoutParams();
                int leftChild = mLeft + lp.leftMargin;
                int topChild = mTop + lp.topMargin;
                int rightChild = leftChild + childView.getMeasuredWidth()；
                int bottomChild = topChild + childView.getMeasuredHeight();
                //四个参数分别表示View的左上角和右下角
                childView.layout(leftChild, topChild, rightChild, bottomChild);
                mLeft += lp.leftMargin + childView.getMeasuredWidth() + lp.rightMargin;
            }
            mLeft = 0;
            mTop += lineMaxHeight;
        }
    }

最终我们可以在 XML 中使用此自定义控件：

[![onLayout_02.png](https://z3.ax1x.com/2021/08/11/fUzLX8.png)](https://imgtu.com/i/fUzLX8)

一个简易的 FlowLayout 运行效果如下所示，剩下的就是和 UI 同事合作，修改 FlowLayout 内部 TextView 的样式，将界面调整到最佳显示状态。

[![onLayout_03.png](https://z3.ax1x.com/2021/08/11/faSljK.png)](https://imgtu.com/i/faSljK)

## 总结

本课时介绍了自定义 View 的几个知识点，要自定义一个控件主要包含几个方法。
onDraw：主要负责绘制 UI 元素；
onMeasure：主要负责测量自定义控件具体显示的宽高；
onLayout：主要是在自定义 ViewGroup 中复写，并实现子 View 的显示位置，并在其中介绍了自定义属性的使用方法。






























































































































































































