# RecyclerView 源码分析 -（一、绘制原理 ）

# 绘制流程

RecyclerView 本质依旧还是一个View 对象，它还是得需要相应的绘制流程三板斧：**onMeasure、onLayout、onDraw** .

那我们依旧按着流程来分析/拆解 它的神秘面纱.

# 一、onMeasure() 先从它开始：

我们先将该部分的代码进行拆解：

```java
    @Override
    protected void onMeasure(int widthSpec, int heightSpec) {
  
        if (mLayout == null) {
		      // case 1, 如果mLayout 为空的话. 就直接执行 defaultOnMeasure.
		      return;
        }

        if (mLayout.mAutoMeasure) {
	        // case 2, 如果mLayout.mAutoMeasure为true 则使用 (如 LinearLayoutManager 该值就为 true.)					
        } else 
	        // case 3, 如果涵盖Layout，并且Layout.mAutoMeasure=false 的话.	        
        }
    }
```

> **mLayout 对应的类： LayoutManager.  通过setLayoutManager函数进行填充 / 设置**
> 

我们从以上代码可以看到整个onMeasure() 函数，拆解出了三个部分的测量策略：

- case 1：未设置 LayoutManager
- case 2:  设置了LayoutManager 并且 Layout.mAutoMesure = true
- case 3：设置了LayoutManager 但 Layout.mAutoMeasure = false

基于以上策略，我们来进行源码进行一部分析。在最后进行整体总结：

## 1.1 Case 1：mLayout 为空.

```java
// case 1, 如果mLayout 为空的话. 就直接执行 defaultOnMeasure.
		if (mLayout == null) {
		   defaultOnMeasure(widthSpec, heightSpec);
		   return;
		}
		
// ---> 对应函数
    void defaultOnMeasure(int widthSpec, int heightSpec) {
        // calling LayoutManager here is not pretty but that API is already public and it is better
        // than creating another method since this is internal.
        final int width = LayoutManager.chooseSize(widthSpec,
                getPaddingLeft() + getPaddingRight(),
                getMinimumWidth());
        final int height = LayoutManager.chooseSize(heightSpec,
                getPaddingTop() + getPaddingBottom(),
                getMinimumHeight());

        setMeasuredDimension(width, height);
    }
```

以上代码我可以看到关键部分，LayoutManager.chooseSize()函数. 

```java
        public static int chooseSize(int spec, int desired, int min) {
            final int mode = View.MeasureSpec.getMode(spec);
            final int size = View.MeasureSpec.getSize(spec);
            switch (mode) {
                case View.MeasureSpec.EXACTLY:
                    return size;
                case View.MeasureSpec.AT_MOST:
                    return Math.min(size, Math.max(desired, min));
                case View.MeasureSpec.UNSPECIFIED:
                default:
                    return Math.max(desired, min);
            }
        }
```

该函数实现其实比较简单，主要是是基于Spec获取了对应的尺寸、模式。获取到RecyclerView 宽高.

最后基于 **chooseSize** 函数获取到的宽高，将其通过setMeasuredDimension()函数赋值于mMeasureHeight、mMeasureWidth. （View类中逻辑）.

## 1.2 Case 2：mLayout 不为空并且 mAutoMeasure 为 true.

该种类型通过字段定义其实就可以得到很好的理解（由LayoutManager 自测量），但比较有意义的分析在于作者如何将该部分进行抽象：

先见代码：（其实LineageLayoutManager就是 mAutoMeasure = true）.

```java
				// case 2，mAutoMeasure 为 true.        
        if (mLayout.mAutoMeasure) {
            //  第一部分：
            // action 1: 先获取对应的测量模式， 通过LayoutManager.onMeasure() 函数获取初步内容.
            final int widthMode = MeasureSpec.getMode(widthSpec);
            final int heightMode = MeasureSpec.getMode(heightSpec);
            // action 2： 如果为 Exactly就为true。下面部分就进行跳过. 因为它已经是精准的大小了.
            final boolean skipMeasure = widthMode == MeasureSpec.EXACTLY
                    && heightMode == MeasureSpec.EXACTLY;
            // action 3：通过LayoutManager.onMeasure函数来自我进行测量.
            mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
            // 这两个需要看下.特别是skipMeasure.
            if (skipMeasure || mAdapter == null) {
                return;
            }

           //  第二部分：
           ...
        } else {
	        // case 3.....
        }
```

截止到：if (skipMeasure || mAdapter == null ) return ; 的 流程 / 逻辑：

1. 获取宽高对应的测量模式
2. 依据测量模式判断是否应该跳出 后续代码？
3. 调用代码来自测量

以上这种场景下，我们其实应该重点来看下两个部分，第一个部分是skipMeasure 与 mLayout.onMeasure()。

### 问题一：

mLayout.onMeasure() 怎么做到到的？ 

答：

我们可以看到在官方的三种 LayoutManager（LinearLayoutManager、GridLayoutManager、StaggeredGridLayoutManager) 都没有复写对应的方法，那么则参考看父类LayoutManager实现：

```java
        public void onMeasure(Recycler recycler, State state, int widthSpec, int heightSpec) {
            mRecyclerView.defaultOnMeasure(widthSpec, heightSpec);
        }
```

可以看到，最终依旧还是交给了RecyclerView.defaultOnMeasure() 函数来实现.

前面已经提到过了，它只是获取自身的宽高及模式。 到目前为止其实还没有进行真正意义上的测量.

### 问题二：

为什么当宽高的MeasureSpec.EXACTLY  为空，就会导致后续的代码被跳过（return）?  mAdater为空，同样是无法测量子布局的大小所以跳过.

答：

很简单，已经由父类设置了精确的宽高大小，无需要子类的大小去判断它自身了。 

在通过defaultOnMeasure() 的流程中就是确定了其自身的大小.

OK，截止到目前，我们已经看完 onMeasure的第一部分。接着往下看:

```java
        if (mLayout.mAutoMeasure) {
            // 第一部分：
            ...
            
						// 第二部分：
            if (mState.mLayoutStep == State.STEP_START) {
                // todo。
                dispatchLayoutStep1();
            }

            
            // set dimensions in 2nd step. Pre-layout should happen with old dimensions for
            // consistency
            mLayout.setMeasureSpecs(widthSpec, heightSpec);
            mState.mIsMeasuring = true;
            dispatchLayoutStep2();

            // now we can get the width and height from the children.
            mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);

            // if RecyclerView has non-exact width and height and if there is at least one child
            // which also has non-exact width & height, we have to re-measure.
            if (mLayout.shouldMeasureTwice()) {
                mLayout.setMeasureSpecs(
                        MeasureSpec.makeMeasureSpec(getMeasuredWidth(), MeasureSpec.EXACTLY),
                        MeasureSpec.makeMeasureSpec(getMeasuredHeight(), MeasureSpec.EXACTLY));
                mState.mIsMeasuring = true;
                dispatchLayoutStep2();
                // now we can get the width and height from the children.
                mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);
            }
        } else {
	        // case 3.....
        }
```

中间给个小插曲，其实dispatchLayoutStep 有三个方法： dispatchLayoutStep1、dispatchLayoutStep2、dispatchLayoutStep3

| State.mLayoutStep | dispatchLayoutStep | 含义说明 |
| --- | --- | --- |
| STEP_START | dispatchLayoutStep1 | STEP_START 是 State.mLayoutStep的默认值，执行完 dispatchLayoutStep1后会将该状态设置为：STEP_LAYOUT |
| STEP_LAYOUT | dispatchLayoutStep2 | 表明处于Layout阶段，调用dispatchLayoutStep2 对RecyclerView的子View 进行layout，执行完之后会将该状态设置为 STEP_ANIMATIONS |
| STEP_ANIMATIONS | dispatchLayoutStep3 | 表明处于执行动画阶段，调用dispatchLayoutStep3 之后会将该状态再次变为STEP_START |

### dispatchLayoutStep1()

其中可以看到。它在判断状态为 Start 时，将会执行 - dispatchLayoutStep1();

```java
    private void dispatchLayoutStep1() {
        ...
        // 处理适配器更新并设置动画标志.
        processAdapterUpdatesAndSetAnimationFlags();
        ...

        // 如果需要运行简单动画.
        if (mState.mRunSimpleAnimations) {
            // Step 0: Find out where all non-removed items are, pre-layout
            // 遍历所有的子视图.
            int count = mChildHelper.getChildCount();
            for (int i = 0; i < count; ++i) {
                final ViewHolder holder = getChildViewHolderInt(mChildHelper.getChildAt(i));
                // 跳过需要忽略的视图或无视且没有稳定ID的视图.
                if (holder.shouldIgnore() || (holder.isInvalid() && !mAdapter.hasStableIds())) {
                    continue;
                }
                // 记录预布局信息.
                final ItemHolderInfo animationInfo = mItemAnimator
                        .recordPreLayoutInformation(mState, holder,
                                ItemAnimator.buildAdapterChangeFlagsForAnimations(holder),
                                holder.getUnmodifiedPayloads());
                mViewInfoStore.addToPreLayout(holder, animationInfo);
                // 如果需要跟踪旧的更改持有者.
                if (mState.mTrackOldChangeHolders && holder.isUpdated() && !holder.isRemoved()
                        && !holder.shouldIgnore() && !holder.isInvalid()) {
                    long key = getChangedHolderKey(holder);
                    // This is NOT the only place where a ViewHolder is added to old change holders
                    // list. There is another case where:
                    //    * A VH is currently hidden but not deleted
                    //    * The hidden item is changed in the adapter
                    //    * Layout manager decides to layout the item in the pre-Layout pass (step1)
                    // When this case is detected, RV will un-hide that view and add to the old
                    // change holders list.
                    mViewInfoStore.addToOldChangeHolders(key, holder);
                }
            }
        }
        // 如果需要运行预测性动画.
        if (mState.mRunPredictiveAnimations) {
            // Step 1: run prelayout: This will use the old positions of items. The layout manager
            // is expected to layout everything, even removed items (though not to add removed
            // items back to the container). This gives the pre-layout position of APPEARING views
            // which come into existence as part of the real layout.

            // Save old positions so that LayoutManager can run its mapping logic.
            // 保存旧的位置.
            saveOldPositions();
            final boolean didStructureChange = mState.mStructureChanged;
            mState.mStructureChanged = false;
            // temporarily disable flag because we are asking for previous layout
            // 临时禁用结构更改标志，运行布局.
            mLayout.onLayoutChildren(mRecycler, mState);
            mState.mStructureChanged = didStructureChange;

            // 遍历所有子视图.
            for (int i = 0; i < mChildHelper.getChildCount(); ++i) {
                final View child = mChildHelper.getChildAt(i);
                final ViewHolder viewHolder = getChildViewHolderInt(child);
                // 跳过需要忽略的视图.
                if (viewHolder.shouldIgnore()) {
                    continue;
                }
                // 如果视图不在预布局中.
                if (!mViewInfoStore.isInPreLayout(viewHolder)) {
                    int flags = ItemAnimator.buildAdapterChangeFlagsForAnimations(viewHolder);
                    boolean wasHidden = viewHolder
                            .hasAnyOfTheFlags(ViewHolder.FLAG_BOUNCED_FROM_HIDDEN_LIST);
                    if (!wasHidden) {
                        flags |= ItemAnimator.FLAG_APPEARED_IN_PRE_LAYOUT;
                    }
                    // 记录预布局的信息.
                    final ItemHolderInfo animationInfo = mItemAnimator.recordPreLayoutInformation(
                            mState, viewHolder, flags, viewHolder.getUnmodifiedPayloads());
                    if (wasHidden) {
                        recordAnimationInfoIfBouncedHiddenView(viewHolder, animationInfo);
                    } else {
                        mViewInfoStore.addToAppearedInPreLayoutHolders(viewHolder, animationInfo);
                    }
                }
            }
            // we don't process disappearing list because they may re-appear in post layout pass.
            // 清除旧的位置.
            clearOldPositions();
        } else {
            clearOldPositions();
        }
        // 标记退出布局或滚动状态.
        onExitLayoutOrScroll();
        // 恢复请求布局.
        resumeRequestLayout(false);
        // 更新布局步骤为： Step_Layout.
        mState.mLayoutStep = State.STEP_LAYOUT;
    }
```

其中对应代码

重点代码：

```java
    private void processAdapterUpdatesAndSetAnimationFlags() {
        ...
        boolean animationTypeSupported = mItemsAddedOrRemoved || mItemsChanged;
        mState.mRunSimpleAnimations = mFirstLayoutComplete
                && mItemAnimator != null
                && (mDataSetHasChangedAfterLayout
                        || animationTypeSupported
                        || mLayout.mRequestedSimpleAnimations)
                && (!mDataSetHasChangedAfterLayout
                        || mAdapter.hasStableIds());
        mState.mRunPredictiveAnimations = mState.mRunSimpleAnimations
                && animationTypeSupported
                && !mDataSetHasChangedAfterLayout
                && predictiveItemAnimationsEnabled();
    }
```

- mFirstLayoutComplete ：是用于标记是否完成第一次绘制流程.

所以第一次还为进行绘制的时候，mRunSimpleAnimations、mRunPredictiveAnimations 都不会为 true.

### dispatchLayoutStep2

```java
    private void dispatchLayoutStep2() {
        ...
        mState.mItemCount = mAdapter.getItemCount();
        mState.mDeletedInvisibleItemCountSincePreviousLayout = 0;

        // Step 2: Run layout
        mState.mInPreLayout = false;
        mLayout.onLayoutChildren(mRecycler, mState);

        mState.mStructureChanged = false;
        mPendingSavedState = null;

        // onLayoutChildren may have caused client code to disable item animations; re-check
        mState.mRunSimpleAnimations = mState.mRunSimpleAnimations && mItemAnimator != null;
        mState.mLayoutStep = State.STEP_ANIMATIONS;
        onExitLayoutOrScroll();
        resumeRequestLayout(false);
    }
```

此方法我们重点关注 mLayout.onLayoutChildren(mRecycler, mState) 调用.

这个函数是由LayoutManager的子类实现的，直接决定了子View的布局方式 .

在最后设置了一把State.mLayoutStep = State.STEP_ANIMATIONS.

## onMeasure case3 : LayoutManager没有开启自测量.

```java
if (mHasFixedSize) {
    mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
    return;
}
// custom onMeasure
if (mAdapterUpdateDuringMeasure) {
    eatRequestLayout();
    processAdapterUpdatesAndSetAnimationFlags();

    if (mState.mRunPredictiveAnimations) {
        mState.mInPreLayout = true;
    } else {
        // consume remaining updates to provide a consistent state with the layout pass.
        mAdapterHelper.consumeUpdatesInOnePass();
        mState.mInPreLayout = false;
    }
    mAdapterUpdateDuringMeasure = false;
    resumeRequestLayout(false);
}

if (mAdapter != null) {
    mState.mItemCount = mAdapter.getItemCount();
} else {
    mState.mItemCount = 0;
}
eatRequestLayout();
mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
resumeRequestLayout(false);
mState.mInPreLayout = false; // clear
```

如果 mHasFixedSize为true，（setHasFixedSize 方法可以设置该变量），就直接调用LayoutManager.onMeasure()方法进行测量；

如果mHasFixedSize为false，则先判断是否有数据更新（mAdapterUpdateDuringMeasure 变量），有的话先处理数据更新，再调用LayoutManager.onMeasure方法进行测量.

# 二、onLayout

该方法叫简单：

```java

    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        Trace.beginSection(TRACE_ON_LAYOUT_TAG);
        dispatchLayout();
        Trace.endSection();
        mFirstLayoutComplete = true;
    }
```

重点方法在是dispatchLayout()

```java
    void dispatchLayout() {
        if (mAdapter == null) {
            Log.e(TAG, "No adapter attached; skipping layout");
            // leave the state in START
            return;
        }
        if (mLayout == null) {
            Log.e(TAG, "No layout manager attached; skipping layout");
            // leave the state in START
            return;
        }
        mState.mIsMeasuring = false;
        if (mState.mLayoutStep == State.STEP_START) {
            dispatchLayoutStep1();
            mLayout.setExactMeasureSpecsFrom(this);
            dispatchLayoutStep2();
        } else if (mAdapterHelper.hasUpdates() || mLayout.getWidth() != getWidth()
                || mLayout.getHeight() != getHeight()) {
            // First 2 steps are done in onMeasure but looks like we have to run again due to
            // changed size.
            mLayout.setExactMeasureSpecsFrom(this);
            dispatchLayoutStep2();
        } else {
            // always make sure we sync them (to ensure mode is exact)
            mLayout.setExactMeasureSpecsFrom(this);
        }
        dispatchLayoutStep3();
    }
```

这个方法其实也很简单明了，首先是判断了mAdapter 、mLayout是否为nul，只要为空则直接return.

dispatchLayout 其主要是保证了dispatchLayoutStep1、2、3 一定是完成执行的. 由于上部分文章已经介绍过Step1、Step2。本段我们直接来介绍Step3

```java
private void dispatchLayoutStep3() {
    
    ...
    
    mState.mLayoutStep = State.STEP_START;
  
    ...
   
}
```

其代码内部大多数方法与Step1、Step2非常雷同。 但主要看到的是，它会将mState.mLayoutStep 再次设置为 State.STEP_START.

这样从而保证了 下一次layout时仍会执行dispatchLayout Step1、Step2、Step3. 剩下的工作主要是和item动画、ItemAnimator相关.

> 如果是RecyclerView中开启了自动测量，在measure阶段就已经将View布局好了，如果没有开启自动测量。那么就会在layout 阶段在布局子view.
> 

# 三、Draw方法.

```java
public void draw(Canvas c) {
    super.draw(c);

    final int count = mItemDecorations.size();
    for (int i = 0; i < count; i++) {
        mItemDecorations.get(i).onDrawOver(c, this, mState);
    }
    // TODO If padding is not 0 and clipChildrenToPadding is false, to draw glows properly, we
    // need find children closest to edges. Not sure if it is worth the effort.
    ......
}
```

draw 大致分为三步：

1. 调用super.draw()： 将子View的绘制分发给了View类；在View类draw()方又会回调到RecyclerView.onDraw() 上来.
2. 调用ItemDecorations.onDrawOver：通过这个方法在我们每个Item 上自定义一些装饰.
3. 如果RecyclerView 调用了 setClipToPadding，会实现一些特殊的滑动效果 — 每个itemView可以滑动到padding区域.

# 四、绘制流程补充举例：

LinearLayoutManager.onLayoutChildren()

在讲解ListView的绘制过程中，我们的中心就是layoutChildren方法，讲解了怎么对子View布局，到现在为止我们还没有进入RecyclerView对子View布局讲解，前面描述dispatchLayoutStep2过程中，我们提到了onLayoutChildren 这个函数由LayoutManager的子类实现。

接下来我就用LinearLayoutManager.onLayoutChildren来举例分析：

```java
public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {
    // layout algorithm:
    // 1) by checking children and other variables, find an anchor coordinate and an anchor
    //  item position.
    // 2) fill towards start, stacking from bottom
    // 3) fill towards end, stacking from top
    // 4) scroll to fulfill requirements like stack from bottom.
    // create layout state
    /**
         * 布局算法：
         * 1. 通过检查子变量和其他变量找到一个锚坐标和锚项位置
         * 2. 向开始进行填充，从底部堆叠.
         * 3. 向结束填充，从顶部堆叠.
         * 4. 滚动以满足从底部堆叠的要求
         * 创建布局状态.
         */
    ...
    
    // 第一步
    resolveShouldLayoutReverse();
    
    if (!mAnchorInfo.mValid || mPendingScrollPosition != NO_POSITION ||
            mPendingSavedState != null) {
        mAnchorInfo.reset();
        mAnchorInfo.mLayoutFromEnd = mShouldReverseLayout ^ mStackFromEnd;
        // 计算锚点位置和坐标
        updateAnchorInfoForLayout(recycler, state, mAnchorInfo);
        mAnchorInfo.mValid = true;
    }
    
    ...
    
    // 第二步
    onAnchorReady(recycler, state, mAnchorInfo, firstLayoutDirection);
    detachAndScrapAttachedViews(recycler);
    mLayoutState.mInfinite = resolveIsInfinite();
    mLayoutState.mIsPreLayout = state.isPreLayout();
    
    // 第三步
    if (mAnchorInfo.mLayoutFromEnd) {
        // fill towards start
        updateLayoutStateToFillStart(mAnchorInfo);
        mLayoutState.mExtra = extraForStart;
        fill(recycler, mLayoutState, state, false);
        
        ...
        
        // fill towards end
        updateLayoutStateToFillEnd(mAnchorInfo);
        mLayoutState.mExtra = extraForEnd;
        mLayoutState.mCurrentPosition += mLayoutState.mItemDirection;
        fill(recycler, mLayoutState, state, false);
        
        ...
        
    } else {
        // fill towards end
        updateLayoutStateToFillEnd(mAnchorInfo);
        mLayoutState.mExtra = extraForEnd;
        fill(recycler, mLayoutState, state, false);
        
        ...
        
        // fill towards start
        updateLayoutStateToFillStart(mAnchorInfo);
        mLayoutState.mExtra = extraForStart;
        mLayoutState.mCurrentPosition += mLayoutState.mItemDirection;
        fill(recycler, mLayoutState, state, false);
        
        ...
        
    }

    ...
    
}
```

该方法其实在开始的注解就很清晰的描述了它要做的事情：

1. 确认锚点信息，找到一个锚点坐标与锚点第一个Item。 （如果是LinearLayoutManager，相当于找到当前界面内第一个View。与第一个View的坐标点.）
2. 根据锚点信息，进行填充
3. 填充完成后，如果还有剩余空间，再次填充一次。（哪怕是1px）

## 第一步：确认锚点信息

resolveShouldLayoutReverse 方法：

```java
		private void resolveShouldLayoutReverse() {
        // A == B is the same result, but we rather keep it readable
        if (mOrientation == VERTICAL || !isLayoutRTL()) {
            mShouldReverseLayout = mReverseLayout;
        } else {
            mShouldReverseLayout = !mReverseLayout;
        }
    }
```

默认情况下 mReverseLayout 为 false，是不会倒着绘制的。 手动调用setReverseLayout方法可以改变mReverseLayout的值.

紧接着就到了updateAnchorInfoForLayout函数 计算锚点信息，这里对锚点类型进行简单讲解：

```java
class AnchorInfo {
        int mPosition;
        int mCoordinate;
        boolean mLayoutFromEnd;
        boolean mValid;

        AnchorInfo() {
            reset();
        }

        void reset() {
            mPosition = NO_POSITION;
            mCoordinate = INVALID_OFFSET;
            mLayoutFromEnd = false;
            mValid = false;
        }
        ...
}
```

AnchorInfo中有四个重要的成员变量，mPosition和 mCoordinate易懂；mValid的默认值是false，一次测量之后设置为true，onLayout完成后会回调执行reset方法，变成为false.

mLayoutFromEnd 变量在计算锚点过程中有如下赋值：

```java
mLayoutFromEnd = mShouldReverseLayout ^ mStackFromEnd
```

mShouldReverseLayout 默认是false，mStackFromEnd 默认是false，除非手动调用setStackFromEnd()方法，两个变量都是false，异或计算之后还是false.

紧接着就是真正开始计算updateAnchorInfoForLayout函数了.

```java
    private void updateAnchorInfoForLayout(RecyclerView.Recycler recycler, RecyclerView.State state,
            AnchorInfo anchorInfo) {
        // 第一种计算方式.
        if (updateAnchorFromPendingData(state, anchorInfo)) {
            if (DEBUG) {
                Log.d(TAG, "updated anchor info from pending information");
            }
            return;
        }
        // 第二种计算方式.
        if (updateAnchorFromChildren(recycler, state, anchorInfo)) {
            if (DEBUG) {
                Log.d(TAG, "updated anchor info from existing children");
            }
            return;
        }
        if (DEBUG) {
            Log.d(TAG, "deciding anchor info for fresh state");
        }
        // 第三种计算方式.
        anchorInfo.assignCoordinateFromPadding();
        anchorInfo.mPosition = mStackFromEnd ? state.getItemCount() - 1 : 0;
    }
```

1. 第一种计算方式：如果存在未决的滚动位置或保存的状态，从该数据更新锚点信息并返回true.
2. 第二种计算方式：根据子View来更新锚点信息，如果一个子View有焦点，则根据其来计算锚点信息；如果一个子View没有锚点，则根据布局方向选取第一个View来计算锚点信息计算.
3. 第三种计算方式：前两种都未采用，则采用默认的第三中计算方式.

## 第二步：回收子View.

第二步是调用detechAndScrapAttachedViews() 方法对所有的ItemView进行回收，这部分的内容属于RecyclerView 缓存机制的部分，后面解释缓存的时候再说;

## 第三步：子View填充 fill().

第三步便是子View的内容填充了，首先是根据mAnchorInfo，mLayoutFromEnd判断是否逆向填充，无论是正向还是逆向。都调用了至少两次fill()方法来进行填充；

如果是正向填充的话先向下填充，再往上填充；两次fill之后，如果还有剩余空间则还会在调用一次fill进行填充。 我们来绘制一张图来了解锚点与fill之间的关系：

及对应的代码：

```java
int fill(RecyclerView.Recycler recycler, LayoutState layoutState,
        RecyclerView.State state, boolean stopOnFocusable) {
    
    ...
    
    while ((layoutState.mInfinite || remainingSpace > 0) && layoutState.hasMore(state)) {
        ...
        
        layoutChunk(recycler, state, layoutState, layoutChunkResult);
        
        ...
        
    }
    
    ...
    
}
```

fill中真正填充子Vie的方法是layoutChunk()，再来看下 layoutChunk 函数实现：

```java
void layoutChunk(RecyclerView.Recycler recycler, RecyclerView.State state,
        LayoutState layoutState, LayoutChunkResult result) {
    // 第一步
    View view = layoutState.next(recycler);
    
    ...
    
    // 第二步
    if (layoutState.mScrapList == null) {
        if (mShouldReverseLayout == (layoutState.mLayoutDirection
                == LayoutState.LAYOUT_START)) {
            addView(view);
        } else {
            addView(view, 0);
        }
    } else {
        if (mShouldReverseLayout == (layoutState.mLayoutDirection
                == LayoutState.LAYOUT_START)) {
            addDisappearingView(view);
        } else {
            addDisappearingView(view, 0);
        }
    }
    
    // 第三步
    measureChildWithMargins(view, 0, 0);
    ...
    
    // 第四步
    layoutDecoratedWithMargins(view, left, top, right, bottom);

    ...
}
```

其layoutChunk函数中主要工作分为以下几步：

1. 调用LayoutState的next方法获取一个ItemView，next方法的参数是Recycler对象，Recycler正是RecyclerView的缓存核心实现，可见RecyclerView中的缓存机制从此处开始。后续分析缓存时再来具体查看：
2. 如果RecyclerView 是第一次布局children（layoutState.mScrapList == null )，会调用addView() 方法将View添加到RecyclerView中
3. 调用measureChildWithMargins方法，测量每个ItemView的宽高。这里会考虑了margin属性和ItemDecoration的offset.
4. 调用了layoutDecorateWithMargins对子View完成了布局.

其中很多细节并未展开描述，后续分析缓存机制时再进行讲解.

# 总结：

1. RecyclerView的 measure过程分为 三种情况，每种情况都有执行过程。一般情况下会走自动测量流程.
2. 自动测量根据mState.LayoutStep状态值，调用不同的dispatchLayoutStep 方法.
3. layout过程也根据mState.mLayoutStep状态来调用不同的dispatchLayoutStep方法
4. draw 过程大致可以分为三步：
    1. 调用super.draw()： 将子View的绘制分发给了View类；在View类draw()方又会回调到RecyclerView.onDraw() 上来.
    2. 调用ItemDecorations.onDrawOver：通过这个方法在我们每个Item 上自定义一些装饰.
    3. 如果RecyclerView 调用了 setClipToPadding，会实现一些特殊的滑动效果 — 每个itemView可以滑动到padding区域.
5. 布局子View 先获取锚点信息，再根据锚点信息和布局方向来进行子View 填充.

基于以上5个步骤，则完成了RecyclerView的 测量、布局、绘制三板斧.