# RecyclerView 源码分析 -（二、缓存策略）

# 一、概述：

我们都知道RecyclerView 无论如何都不会导致OOM的发生，而这个背后依靠的是其本身强大的缓存、复用策略. 那么导致是如何实现的？ 我们更应该去深入了解.

官方介绍：为大量数据提供优先展示窗口的灵活视图。 

### 为了后续的文章理解，我们先了解几个方法的含义：

| 方法 | 对应的Flag | 含义 | 出现场景 |
| --- | --- | --- | --- |
| isInvalid() | FLAG_INVALID | ViewHolder的数据是无效的 | 1. 调用adapter的setAdaptet()
2. adapter调用了 notifyDataSetChanged()
3. 调用 RecyclerView的 invalidateItemDecorations() |
| isRemoved() | FLAG_REMOVED | ViewHolder已经被移除，源数据被移除了部分数据 | adapter调用了notifyItemRemoved() |
| isUpdated() | FLAG_UPDATE | item的ViewHolder 数据信息过时了，需要重新绑定数据 | 1. 上述 isInvalid()的三种情况都会
2. 调用Adapter的onBinderViewHolder()
3. 调用Adapter的notifyItemChanged() |
| isBound() | FLAG_BOUND | ViewHolder 已经绑定了某个位置的Item上，数据是有效的. | 调用了onBinderViewHolder() 函数. |

# 二、Recycler的几级缓存

RecyclerView 不会像ListView 那样 if(contentView == null) {} else {} 处理复用的逻辑，它回收复用是由Recycler来负责的，它是负责管理 scrapped（废弃）或者 detached（分离）的视图（ViewHolder）以便重复使用。

想要了解RecyclerView的回收复用原理，那么首先了解 Recycler的几个结构：

```java
    public final class Recycler {
        final ArrayList<ViewHolder> mAttachedScrap = new ArrayList<>();
        ArrayList<ViewHolder> mChangedScrap = null;

        final ArrayList<ViewHolder> mCachedViews = new ArrayList<ViewHolder>();

        private final List<ViewHolder>
                mUnmodifiableAttachedScrap = Collections.unmodifiableList(mAttachedScrap);

        private int mRequestedCacheMax = DEFAULT_CACHE_SIZE;
        int mViewCacheMax = DEFAULT_CACHE_SIZE;

        RecycledViewPool mRecyclerPool;

        private ViewCacheExtension mViewCacheExtension;
        
        ....
   }
```

在Recycler 中设置了四级缓存池，按照使用的优先顺序，给出以下表格：

| 缓存 | 涉及对象 / 成员 | 作用 | 重新创建视图 | 重新绑定数据 |
| --- | --- | --- | --- | --- |
| 一级缓存 | 1. mAttachedScrap                        2. mChangedScrap | 缓存屏幕中可见范围的ViewHolder | false | false |
| 二级缓存 | mCachedViews | 缓存滑动时即将与RecyclerView | false | false |
| 三级缓存 | mViewCacheExtension | 开发者自行实现的缓存 | - | - |
| 四级缓存 | mRecyclerPool | ViewHolder缓存 池，本质上是一个SparseArray，其中key 是 ViewType(Int 类型），value 存放的是ArrayList<ViewHolder>，默认每个ArrayList 中最多存放5个ViewHolder. | false | true |

按照顺序：Scrap → CachedViews → ViewCacheExtension → RecyclerViewPool

我们依次在重点介绍下：

- AttachedScrap：不参与滑动的回收复用，只保存重新布局时从RecyclerView 分离的Item 无效、未移除、未更新的Holder。因为RecyclerView 在onLayout的时候会先把children全部抹除掉，再重新添加进入. mAttachedScrap临时保存这些Holder复用.
- ChangedScrap：mChangedaScrap 和 mAttachedScrap类似，不参与滑动时的回收复用，知识用作临时保存变量，它指挥负责保存重新布局时发生变化的item无效、未移除的holder，那么会重走Adapter绑定数据的方法.
- CachedViews：用于保存最新被移除（remove）的ViewHolder，已经和RecyclerView 分离的视图；它的作用是滚动的回收复用时如果需要新的ViewHolder 时，精准匹配（根据 position / id判断）是不是原来被移除的那个Item；如果是，则直接返回ViewHolder使用。不需要重新绑定数据.如果不是则不返回再去RecyclerViewPool中去找Holder 实例返回，并重新绑定数据。 这一级的缓存是有容量限制的，最大数量为2. （这一级缓存是使用率最高的）
- ViewCacheExtension：RecyclerView 给开发者预留的缓存池，开发者可以自己拓展回收池，一般不会用到，用RecyclerView 系统自带的已经足够了.
- RecyclerViewPool：是一个最终回收站，真正存放着被着被表示废弃（其他池都不愿意回收）的ViewHolder的缓存池，如果上述 mAttachedScrap、mChangedScrap、mCachedViews、mViewCacheExtension都找不到ViewHolder的情况下，就会从mRecyclerPool 返回一个废弃的ViewHolder实例，但是这里的ViewHolder 是已经被抹除数据的，没有任何绑定的痕迹，需要重新绑定数据。 它是根据itemType来存储的，是以SparseArray 嵌套一个ArrayList的形式保存ViewHolder的.

接着我们来详细分析一个各个缓存池：

## 2.1、一级缓存池 - Scrap

Scrap 是RecyclerView最轻量级的缓存，包括mAttachedScrap 和 mChangedScrap，它不参与列表滚动时的回收复用，作为重新布局的临时缓存，它的作用是：

- 缓存当界面重新布局前 和 界面重新布局后都出现的ViewHolder。 这些ViewHolder是无效、未移除、未标记的。在这些无效、未移除、未标记的ViewHolder之中;
- AttachedScrap 负责保存其中没有改变的ViewHolder；
- ChangedScrap 负责其他有所改变的ViewHolder;

AttachedScrap、ChangedScrap 只是分工合作保存不同的ViewHolder 而已

> 注意：Scrap只是作为布局的临时缓存，它和滑动时的缓存没有任何关系，它的detach 和 atach只是临时存在于布局过程中。布局结束时，Scrap列表应该是空的，缓存的数据要么重新布局出来，要么被清空； 总之在布局结束后Scrap列表不应该存在任何东西.
> 

看Scrap 复用场景.

![RecyclerView - Scrap.jpg](RecyclerView%20%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%20-%EF%BC%88%E4%BA%8C%E3%80%81%E7%BC%93%E5%AD%98%E7%AD%96%E7%95%A5%EF%BC%89%201b7f25c0fa0d80488713cc18845e97cd/RecyclerView_-_Scrap.jpg)

在一个手机屏幕中将ItemB删除，并且调用notifyItemRemoved()方法，如果Item是无效并且被移除的就会回收到其他的缓存，否则都是缓存到Scrap中，那么mAttachedScrap 和 mChangedScrap 会分别存储itemView，ItemA没有任何变化。

存储到mAttachedScrap中，ItemB虽然被移出了，但是还是有效，也被存储到mAttachedScrap中（但是会被标记 REMOVED，之后会被移除）；ItemC和ItemD发生了变化，位置网上移动了会被存储到mChangedScrap中。

删除时，ABCD都会进入Scrap中；删除后 ACD都会回来，A没有任何变化，CD只是发生了位置上变化。内容没有发生变化.

RecyclerViw的局部刷新就是依赖Scrap的临时缓存，当我们通过notifyItemRemoved()、notifyItemChanged() 通知Item发生变化的时候，通过mAttachedScrap 缓存没有发生变化的ViewHolder，其他的则由mChangedScrap缓存，添加ItemView的时候快速从里面取出，完成局部刷新。

注意，如果我们是使用notifyDataChanged() 来通知RecyclerView刷新，屏幕上的ItemView被标记为FLAG_INVALID并且未被移除，所以不会使用Scrap缓存，而且直接扔到 CacheView或者RecyclerViewPool 池中，回来的时候重新走一次绑定数据.

> 注意：ItemE 并没有出现在屏幕中，它不属于Scrap管辖的范围，Scrap只会换在屏幕中已经加载出来的ItemView的Holder.
> 

## 2.2 二级缓存池 - CacheView

CacheView 用于RecyclerView 列表位置产生变动时，对刚刚移除屏幕的view进行回收复用。根据position / id 来精准匹配是不是原来的Item，如果是则直接返回使用，不需要重新绑定数据；如果不是则去RecyclerViewPool 找到Holder 实例返回，并且重新绑定数据。

这里面一个比较值得关注的问题就是 CacheView 容量大小只有：2.缓存一个新的ViewHolder时，如果超出最大限制，那么会将CacheView缓存的第一个数据 添加到RecyclerViewPool中后 再CacheView中移除掉. 最后才会将新的ViewHolder 添加进来。

我们在滑动RecyclerView 的时候，Recycler 会不断的缓存刚刚移除屏幕不可见的View到 CacheView中，CacheView到达上限时又会不断替换CacheView中旧的ViewHolder，将它们扔到RecyclerViewPool之中。

如果一直朝一个方向滑动，CacheView并没有在效率上产生帮助，它只是把后面滑过的ViewHolder 缓存起来，如果经常来回滑动，那么从CacheView根据对应的Item直接调用，不需要重新绑定数据，将会得到很好的利用.

看Cache复用场景：

![RecycerView - CacheView.jpg](RecyclerView%20%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%20-%EF%BC%88%E4%BA%8C%E3%80%81%E7%BC%93%E5%AD%98%E7%AD%96%E7%95%A5%EF%BC%89%201b7f25c0fa0d80488713cc18845e97cd/RecycerView_-_CacheView.jpg)

从图中可以看出，CacheView 缓存刚刚变为不可见的View， 如果当前View再次进入屏幕中的时候，进行精准匹配，这个itemView还是之前的itemView，那么会从CacheView中获取到ViewHolder进行复用。 如果一直向某一个方向滑动，那么CacheView 将会不断替换缓存里面的ViewHolder（CacheView 最多只能存储2个），将替换掉的ViewHolder 先放到 RecyclerViewPool 中，在CacheView 中拿不到复用的ViewHolder，那么最后只能去 RecyclerViewPool 中获取。

## 2.3 三级缓存 - ViewCacheExtension

ViewCacheExtension 是缓存拓展的帮助类，额外提供了一层缓存池给开发者。开发者根据实际情况来定是否使用ViewCacheExtension 增加了一层缓存池，Recycler 首先去Scrap 和 CacheView中寻找复用View，如果没有就会去ViewCacheExtension中去寻找，如果还是没有寻找到。

那么就会去RecyclerViewPool中寻找复用ViewHolder.

## 2.4 四级缓存 - RecyclerViewPool

在Scrap、CacheView、ViewCacheExtension 都不愿意回收的时候，都会丢到RecyclerViewPool 中回收，所以RecyclerViewPool 是Recycler 的终极缓存.

RecyclerView 实际是以SparseArray 嵌套一个ArrayList的形式来保存ViewHolder，因为RecyclerViewPool 保存的ViewHolder 是以ViewType来区分的。 这样方便不同的ItemType保存不同的ViewHolder.

它在回收的时候只是回收该ViewType的ViewHolder对象，并没有保存原来的数据信息，在复用的时候需要走onBinderViewHolder()函数来重新绑定数据.

我们来看下RecyclerViewPool的结构：

```java
public static class RecycledViewPool {
        private static final int DEFAULT_MAX_SCRAP = 5;

        static class ScrapData {
            ArrayList<ViewHolder> mScrapHeap = new ArrayList<>();
            int mMaxScrap = DEFAULT_MAX_SCRAP;
            long mCreateRunningAverageNs = 0;
            long mBindRunningAverageNs = 0;
        }
        SparseArray<ScrapData> mScrap = new SparseArray<>();

        private int mAttachCount = 0;
        ...
}
```

可以看出，RecyclerViewPool 中定义了SparseArray<ScrapData> mScrap，它是一个根据不同的 itemType来保存静态类 ScrapData对象的SparseArray，ScrapData中包含了 ArrayList<ViewHolder> mScrapHeap，mScrapHeap 是专门用来在RecyclerViewPool中按ViewType来保存 VIewHolder的ArrayList.

缓存池定义了默认的缓存大小 DEFAULT_MAX_SCRAP = 5，这个数量不是说整个缓存池只能缓存这么多ViewHolder，而是不同的ItemType的独立ArrayList 只能存放五个ViewHolder. （我看到有文章说只能支持五个itemType，反正我目前在源码中没有发现这个逻辑）.

基于以上，我们可以看到，在滑动过程中。 如果开发者没有设置ViewCacheExtension的话，实际只有两级缓存会被使用。.

# 三、源码解析（回收和复用）

基于上其实更多是理论，我们来结合实际源码观看。

## 3.1 回收流程：

基于LinearLayoutManger， 看来 onLayoutChildren()

```java
    @Override
    public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {
        if (mPendingSavedState != null || mPendingScrollPosition != RecyclerView.NO_POSITION) {
            if (state.getItemCount() == 0) {
                removeAndRecycleAllViews(recycler);//移除所有子View
                return;
            }
        }
        ensureLayoutState();
        mLayoutState.mRecycle = false;//禁止回收
        //颠倒绘制布局
        resolveShouldLayoutReverse();
        onAnchorReady(recycler, state, mAnchorInfo, firstLayoutDirection);

        //暂时分离已经附加的view，即将所有child detach并通过Scrap回收
        detachAndScrapAttachedViews(recycler);
    }
```

在onLayoutChildren() 布局的时候，先根据实际情况是否需要 removeAndRecycleAllViews() 移除所有的子View，那些ViewHolder 不可用； 然后通过detachAndScrapAttachedViews() 暂时分离已经附加的ItemView，缓存到View中.

我们来重点看下 detachAndScrapAttachedViews 的代码，它的作用就是会将当前屏幕所有的item与屏幕分离，将他们从RecyclerView 的布局拿下来，保存到List中。 在重新布局时，再将ViewHolder重新一个个放到位置上去。

将屏幕上的ViewHolder从RecyclerView的布局中拿下来后，存放在Scrap中，Scrap包括mAttachedScrap 和 mChangedScrap，它们是一个List. 用来保存从RecyclerView布局中拿下来的ViewHolder 列表，detachAndScrapAttachedViews() 只会在onLayoutChildren()中调用，只有在布局的时候，才会把ViewHolder detach 掉。然后再add 进来重新布局，但是大家需要注意，Scrap只是保存从 RecyclerView 布局中当前屏幕显示的Item的ViewHolder，不参与回收调用，单纯是为了先从RecylcerView中拿来在重新布局上去.

```java
        public void detachAndScrapAttachedViews(Recycler recycler) {
            final int childCount = getChildCount();
            for (int i = childCount - 1; i >= 0; i--) {
                final View v = getChildAt(i);
                scrapOrRecycleView(recycler, i, v);
            }
        }

        private void scrapOrRecycleView(Recycler recycler, int index, View view) {
            final ViewHolder viewHolder = getChildViewHolderInt(view);
            // 如果该视图应被忽略，则直接return
            if (viewHolder.shouldIgnore()) {
                if (DEBUG) {
                    Log.d(TAG, "ignoring view " + viewHolder);
                }
                return;
            }
            // 如果视图无效且被移除，且适配器没有稳定的ID.
            if (viewHolder.isInvalid() && !viewHolder.isRemoved()
                    && !mRecyclerView.mAdapter.hasStableIds()) {
                // 从布局中移除视图
                removeViewAt(index);
                // 回收该视图持有者.
                recycler.recycleViewHolderInternal(viewHolder);
            } else {
                // 将视图从布局中分离
                detachViewAt(index);
                // 将视图暂存以备用
                recycler.scrapView(view);
                // 通知视图信息存储，视图已经被分离.
                mRecyclerView.mViewInfoStore.onViewDetached(viewHolder);
            }
        }
```

遍历所有View，分离所有已经添加到RecyclerView 的ItemView，Recycler先废弃他们，然后再进缓存列表中拿出来复用.

在scrapOrRecyclerView函数中，进入else分支，可以看到先detachViewAt() 分离视图，然后在通过recycler.scrapView() 缓存到scrap中.

```java
        void scrapView(View view) {
            final ViewHolder holder = getChildViewHolderInt(view);
            // 检查 ViewHolder 是否具有REMOVE 或 INVALID 标志，或视图未更新，或视图可复用.
            if (holder.hasAnyOfTheFlags(ViewHolder.FLAG_REMOVED | ViewHolder.FLAG_INVALID)
                    || !holder.isUpdated() || canReuseUpdatedViewHolder(holder)) {
                // 如果视图无效但未被移除，且适配器没有稳定的ID 抛出异常.
                if (holder.isInvalid() && !holder.isRemoved() && !mAdapter.hasStableIds()) {
                    throw new IllegalArgumentException("Called scrap view with an invalid view."
                            + " Invalid views cannot be reused from scrap, they should rebound from"
                            + " recycler pool.");
                }
                // 设置 ViewHolder 的暂存容器为当前 Recycler，标记为未更改.
                holder.setScrapContainer(this, false);
                // 将 ViewHolder 添加到 mAttachedScrap 集合中.
                mAttachedScrap.add(holder);
            } else {
                // 如果 mChangedScrap 为空，初始化为 新的 ArrayList.
                if (mChangedScrap == null) {
                    mChangedScrap = new ArrayList<ViewHolder>();
                }
                // 设置ViewHolder 的暂存容器为当前 Recycler，标记为 已更改.
                holder.setScrapContainer(this, true);
                // 将ViewHolder 添加到 mChangedScrap之中去.
                mChangedScrap.add(holder);
            }
        }
```

回到scrapOrRecyclerView() 中，进入if() 分支如果ViewHolder是无效、未被移除、未被标记的则放入recyclerViewHolderInternal() 缓存起来。同时removeViewAt() 移除了ViewHolder.

```java
   void recycleViewHolderInternal(ViewHolder holder) {
           ·····
        if (forceRecycle || holder.isRecyclable()) {
            if (mViewCacheMax > 0
                    && !holder.hasAnyOfTheFlags(ViewHolder.FLAG_INVALID
                    | ViewHolder.FLAG_REMOVED
                    | ViewHolder.FLAG_UPDATE
                    | ViewHolder.FLAG_ADAPTER_POSITION_UNKNOWN)) {

                int cachedViewSize = mCachedViews.size();
                if (cachedViewSize >= mViewCacheMax && cachedViewSize > 0) {//如果超出容量限制，把第一个移除
                    recycleCachedViewAt(0);
                    cachedViewSize--;
                }
                 	·····
                mCachedViews.add(targetCacheIndex, holder);//mCachedViews回收
                cached = true;
            }
            if (!cached) {
                addViewHolderToRecycledViewPool(holder, true);//放到RecycledViewPool回收
                recycled = true;
            }
        }
    }
```

如果符合条件，会优先缓存到mCacheViews之中，如果超出mCachedViews 的最大限制 则通过recyclerCacheViewAt() 将CacheView 缓存的第一个数据添加到 RecyclerViewPool后在移除掉。最后才会add()到mCacheViews之中.

剩下不符合条件的则通过addViewHolderToRecycledViewPool() 缓存到RecyclerViewPool中。

```java
        void addViewHolderToRecycledViewPool(ViewHolder holder, boolean dispatchRecycled) {
            clearNestedRecyclerViewIfNotNested(holder);
            holder.itemView.setAccessibilityDelegate(null);
            if (dispatchRecycled) {
                dispatchViewRecycled(holder);
            }
            holder.mOwnerRecyclerView = null;
            getRecycledViewPool().putRecycledView(holder);
        }
```

还有一个就是在填充布局 fill() 的时候，它会回收移除屏幕的View到mCacheViews 或者 RecyclerViewPool之中.

```java
    int fill(RecyclerView.Recycler recycler, LayoutState layoutState,
            RecyclerView.State state, boolean stopOnFocusable) {
        if (layoutState.mScrollingOffset != LayoutState.SCROLLING_OFFSET_NaN) {
              recycleByLayoutState(recycler, layoutState);//回收移出屏幕的view
        }
    }
```

重点来看：recycleByLayoutState()函数

```java
    private void recycleByLayoutState(RecyclerView.Recycler recycler, LayoutState layoutState) {
        // 如果不需要回收视图或视图为无线循环模式则直接返回.
        if (!layoutState.mRecycle || layoutState.mInfinite) {
            return;
        }
        // 根据布局方向，选择起始或者末尾来回收视图.
        if (layoutState.mLayoutDirection == LayoutState.LAYOUT_START) {
            recycleViewsFromEnd(recycler, layoutState.mScrollingOffset);
        } else {
            recycleViewsFromStart(recycler, layoutState.mScrollingOffset);
        }
    }
```

紧接着我们跟踪 recycleViewsFromStart/End 会发现最后调用的函数: recycleView()

```java
        public void recycleView(View view) {
            // This public recycle method tries to make view recycle-able since layout manager
            // intended to recycle this view (e.g. even if it is in scrap or change cache)
            ViewHolder holder = getChildViewHolderInt(view);
            if (holder.isTmpDetached()) {
                removeDetachedView(view, false);
            }
            if (holder.isScrap()) {
                holder.unScrap();
            } else if (holder.wasReturnedFromScrap()) {
                holder.clearReturnedFromScrapFlag();
            }
            recycleViewHolderInternal(holder);
        }
```

回收分离的视图到缓存池中，方便以后重新绑定和复用。 这里又来到了 recycleViewHolderInternal(holder)，和上面一样。 按照优先级缓存：

- mCacheViews
- RecyclerViewPool

那么回收流程就到这里结束了。

## 3.2 复用流程：

itemView 的回收流程分析完成了，那么这些回收的ViewHOlder到底在什么时候，什么地方拿出来使用呢？？？

我们在接着来看LinearLayoutManager.onLayoutChildren()

```java
  @Override
    public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {
        if (mPendingSavedState != null || mPendingScrollPosition != RecyclerView.NO_POSITION) {
            if (state.getItemCount() == 0) {
                removeAndRecycleAllViews(recycler);//移除所有子View
                return;
            }
        }
    
        //暂时分离已经附加的view，即将所有child detach并通过Scrap回收
        detachAndScrapAttachedViews(recycler);
        
        if (mAnchorInfo.mLayoutFromEnd) {
            //描点位置从start位置开始填充ItemView布局
            updateLayoutStateToFillStart(mAnchorInfo);
            fill(recycler, mLayoutState, state, false);//填充所有itemView
           
 			//描点位置从end位置开始填充ItemView布局
            updateLayoutStateToFillEnd(mAnchorInfo);
            fill(recycler, mLayoutState, state, false);//填充所有itemView
            endOffset = mLayoutState.mOffset;
        }else {
            //描点位置从end位置开始填充ItemView布局
            updateLayoutStateToFillEnd(mAnchorInfo);
            fill(recycler, mLayoutState, state, false);
 
            //描点位置从start位置开始填充ItemView布局
            updateLayoutStateToFillStart(mAnchorInfo);
            fill(recycler, mLayoutState, state, false);
            startOffset = mLayoutState.mOffset;
        }
    }
```

回收View 后，紧接着就是填充VIew，上面提到，在重新布局的时候会临时将View 缓存起来。再一个个把ViewHolder 按照正确的位置填充上去。

fill() 就是负责填充 由layoutState定义的给定布局：

```java
 int fill(RecyclerView.Recycler recycler, LayoutState layoutState, RecyclerView.State state, boolean stopOnFocusable) {
        recycleByLayoutState(recycler, layoutState);//回收滑出屏幕的view
        while ((layoutState.mInfinite || remainingSpace > 0) && layoutState.hasMore(state)) {//一直循环，知道没有数据
            layoutChunkResult.resetInternal();
            layoutChunk(recycler, state, layoutState, layoutChunkResult);//添加一个child
            ······
            if (layoutChunkResult.mFinished) {//布局结束，退出循环
                break;
            }
            layoutState.mOffset += layoutChunkResult.mConsumed * layoutState.mLayoutDirection;//根据添加的child高度偏移计算   
        }
     	······
        return start - layoutState.mAvailable;//返回这次填充的区域大小
    }
```

判断当前可见区域还有没有剩余空间，如果有则填充View 上去，核心是通过 while() 循环执行 layoutChunk() 去一个一个填充到屏幕上去. 

说白了就是layoutChunk()完成布局工作的：

```java
void layoutChunk(RecyclerView.Recycler recycler, RecyclerView.State state,
            LayoutState layoutState, LayoutChunkResult result) {
        View view = layoutState.next(recycler);//获取复用的view
        ······
}
```

该方法通过layoutState.next(recycler) 获取到视图。（又进入到了Recycler类的方法中了）

```java
    // LinearLayoutManager.LayoutState类	
    View next(RecyclerView.Recycler recycler) {
        if (mScrapList != null) {
            return nextViewFromScrapList();
        }
        final View view = recycler.getViewForPosition(mCurrentPosition);
        mCurrentPosition += mItemDirection;
        return view;
    }

		// Recycler 类：
    @NonNull
    public View getViewForPosition(int position) {
        return getViewForPosition(position, false);
    }

    View getViewForPosition(int position, boolean dryRun) {
        return tryGetViewHolderForPositionByDeadline(position, dryRun, FOREVER_NS).itemView;
    }
```

tryGetViewHolderForPositionByDeadline() 才是获取 view的方法，它会根据给出的 position / id 去scrap、cache、RecyclerViewPool 乃至去创建一个 ViewHolder.

```java
    @Nullable
    ViewHolder tryGetViewHolderForPositionByDeadline(int position, boolean dryRun, long deadlineNs) {
        ViewHolder holder = null;
        // 0) 如果它是改变的废弃的ViewHolder，在scrap的mChangedScrap找
        if (mState.isPreLayout()) {
            holder = getChangedScrapViewForPosition(position);
            fromScrapOrHiddenOrCache = holder != null;
        }
        // 1)根据position分别在scrap的mAttachedScrap、mChildHelper、mCachedViews中查找
        if (holder == null) {
            holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
        }

        if (holder == null) {
            final int type = mAdapter.getItemViewType(offsetPosition);
            // 2)根据id在scrap的mAttachedScrap、mCachedViews中查找
            if (mAdapter.hasStableIds()) {
                holder = getScrapOrCachedViewForId(mAdapter.getItemId(offsetPosition), type, dryRun);
            }
            if (holder == null && mViewCacheExtension != null) {
                //3)在ViewCacheExtension中查找，一般不用到，所以没有缓存
                final View view = mViewCacheExtension
                        .getViewForPositionAndType(this, position, type);
                if (view != null) {
                    holder = getChildViewHolder(view);
                }
            }
            //4)在RecycledViewPool中查找
            holder = getRecycledViewPool().getRecycledView(type);
            if (holder != null) {
                holder.resetInternal();
                if (FORCE_INVALIDATE_DISPLAY_LIST) {
                    invalidateDisplayListInt(holder);
                }
            }
        }
        //5)到最后如果还没有找到复用的ViewHolder，则新建一个
        holder = mAdapter.createViewHolder(RecyclerView.this, type);
    }
```

这个方法确实做了不少事情，分别去`scrap`、`CacheView`、`ViewCacheExtension`、`RecycledViewPool`中获取ViewHolder，如果没有则创建一个新的ViewHolder返回，我们一步步来分析：

**第一步**：如果是废弃的发生改变的ViewHolder，则在`scrap`的`mChangedScrap`查找视图，通过`position`和`id`分别查找；
这个一般在我们调用adapter的`notifyItemChanged()`方法时，数据发生变化，item缓存在`mChangedScrap`中，后续拿到的ViewHolder需要重新绑定数据。

```java

   ViewHolder getChangedScrapViewForPosition(int position) {
        //通过position        for (int i = 0; i < changedScrapSize; i++) {
            final ViewHolder holder = mChangedScrap.get(i);
            return holder;
        }
        // 通过id        if (mAdapter.hasStableIds()) {
            final long id = mAdapter.getItemId(offsetPosition);
            for (int i = 0; i < changedScrapSize; i++) {
                final ViewHolder holder = mChangedScrap.get(i);
                return holder;
            }
        }
        return null;
    }

```

**第二步**：如果没有找到视图，根据`position`分别在`scrap`的`mAttachedScrap`、`mChildHelper`、`mCachedViews`中查找。在`getScrapOrHiddenOrCachedHolderForPosition(position, dryRun)`这个方法按照以下顺序查找：

- 首先从`mAttachedScrap`中查找，精准匹配有效的ViewHolder；
- 接着在`mChildHelper`中`mHiddenViews`查找隐藏的ViewHolder；
- 最后从我们的一级缓存中`mCachedViews`查找。

```java

    //根据position分别在scrap的mAttachedScrap、mChildHelper、mCachedViews中查找    ViewHolder getScrapOrHiddenOrCachedHolderForPosition(int position, boolean dryRun) {
        final int scrapCount = mAttachedScrap.size();

        // 首先从mAttachedScrap中查找，精准匹配有效的ViewHolder        for (int i = 0; i < scrapCount; i++) {
            final ViewHolder holder = mAttachedScrap.get(i);
            return holder;
        }
        //接着在mChildHelper中mHiddenViews查找隐藏的ViewHolder        if (!dryRun) {
            View view = mChildHelper.findHiddenNonRemovedView(position);
            if (view != null) {
                final ViewHolder vh = getChildViewHolderInt(view);
                scrapView(view);
                return vh;
            }
        }

        //最后从我们的一级缓存中mCachedViews查找。        final int cacheSize = mCachedViews.size();
        for (int i = 0; i < cacheSize; i++) {
            final ViewHolder holder = mCachedViews.get(i);
            return holder;
        }
    }

```

**第三步**：如果没有找到视图，通过`id`在`scrap`的`mAttachedScrap`、`mCachedViews`中查找。在`getScrapOrCachedViewForId()`这个方法按照以下顺序：

- 首先从`mAttachedScrap`中查找，精准匹配有效的ViewHolder；
- 接着从我们的一级缓存中`mCachedViews`查找；

注意：这一步是跟`id`来查找的，与上一步根据`position`查找类似。

```java

  ViewHolder getScrapOrCachedViewForId(long id, int type, boolean dryRun) {
        //在Scrap的mAttachedScrap中查找        final int count = mAttachedScrap.size();
        for (int i = count - 1; i >= 0; i--) {
            final ViewHolder holder = mAttachedScrap.get(i);
            return holder;
        }

        //在一级缓存mCachedViews中查找        final int cacheSize = mCachedViews.size();
        for (int i = cacheSize - 1; i >= 0; i--) {
            final ViewHolder holder = mCachedViews.get(i);
            return holder;
        }
    }

```

**第四步**：在`mViewCacheExtension`中查找，前面提到这个缓存池是由开发者定义的一层缓存策略，`Recycler`并没有将任何view缓存到这里。这里没有定义过，所有找不到对应的view。

```java

if (holder == null && mViewCacheExtension != null) {
        final View view = mViewCacheExtension.getViewForPositionAndType(this, position, type);
        if (view != null) {
            holder = getChildViewHolder(view);
        }
    }

```

**第五步**：从`RecycledViewPool`中查找，上面讲到它是通过`itemType`把ViewHolder的List缓存到`SparseArray`中的，在`getRecycledViewPool().getRecycledView(type)`根据itemType从`SparseArray`获取`ScrapData` ，然后再从里面获取`ArrayList<ViewHolder>`，从而获取到ViewHolder。

```java

    @Nullable    public ViewHolder getRecycledView(int viewType) {
        final ScrapData scrapData = mScrap.get(viewType);//根据viewType获取对应的ScrapData         if (scrapData != null && !scrapData.mScrapHeap.isEmpty()) {
            final ArrayList<ViewHolder> scrapHeap = scrapData.mScrapHeap;
            for (int i = scrapHeap.size() - 1; i >= 0; i--) {
                if (!scrapHeap.get(i).isAttachedToTransitionOverlay()) {
                    return scrapHeap.remove(i);
                }
            }
        }
        return null;
    }

```

**第六步**：如果还没有获取到ViewHolder，则通过`mAdapter.createViewHolder()`创建一个新的ViewHolder返回。

```java

//5)到最后如果还没有找到复用的ViewHolder，则新建一个
  holder = mAdapter.createViewHolder(RecyclerView.this, type)
```

![image.png](RecyclerView%20%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%20-%EF%BC%88%E4%BA%8C%E3%80%81%E7%BC%93%E5%AD%98%E7%AD%96%E7%95%A5%EF%BC%89%201b7f25c0fa0d80488713cc18845e97cd/image.png)

# 四、总结

### 4.1 RecyclerVIew的回收原理

在RecyclerView重新布局`onLayoutChildren()`或者填充布局`fill()`的时候，会先把必要的item与屏幕分离或者移除，并做好标记，保存到list中，在重新布局时，再将ViewHolde拿出来重新一个个放到新的位置上去。

（1）如果是RecyclerView不滚动情况下缓存(比如删除item)，重新布局时，把屏幕上的ViewHolder与屏幕分离下来，存放到`Scrap`中，即发生改变的ViewHolder缓存到`mChangedScrap`中，不发生改变的ViewHolder存放到`mAttachedScrap`中；剩下ViewHolder的会按照`mCachedViews`>`RecycledViewPool`的优先级缓存到`mCachedViews`或者`RecycledViewPool`中。

（2）如果是RecyclerVIew滚动情况下缓存(比如滑动列表)，在滑动时填充布局，先移除滑出屏幕的item，第一级缓存`mCachedViews`优先缓存这些ViewHolder，但是`mCachedViews`最大容量为2，当`mCachedViews`满了以后，会利用先进先出原则，把旧的ViewHolder存放到`RecycledViewPool`中后移除掉，腾出空间，再将新的ViewHolder添加到`mCachedViews`中，最后剩下的ViewHolder都会缓存到终极回收池`RecycledViewPool`中，它是根据itemType来缓存不同类型的`ArrayList<ViewHolder>`，最大容量为5。

### 4.2 RecyclerVIew的复用原理

至此，已经有五个缓存RecyclerView的池子，`mChangedScrap`、`mAttachedScrap`、`mCachedViews`、`mViewCacheExtension`、`mRecyclerPool`，除了`mViewCacheExtension`是系统提供给开发者拓展的没有用到之外，还有四个池子是参与到复用流程中的。

**（1）当RecyclerView要拿一个复用的ViewHolder时，如果是预加载，则会先去`mChangedScrap`中精准查找(分别根据`position`和`id`)对应的ViewHolder，如果有就返回**；

**（2）如果没有就再去`mAttachedScrap`和`mCachedViews`中精确查找(先`position`后`id`)是不是原来的ViewHolder，如果是说明ViewHolder是刚刚被移除的**；

**（3）如果不是，则最终去`mRecyclerPool`找，如果`itemType`类型匹配对应的ViewHolder，那么返回实例，让它重新绑定数据**；

**（4）如果`mRecyclerPool`也没有返回ViewHolder才会调用`createViewHolder()`重新去创建一个**。

这里需要注意：在`mChangedScrap`、`mAttachedScrap`、`mCachedViews`中拿到的ViewHolder都是精准匹配，但是`mChangedScrap`的是发生了变化的，需要调用`onBindViewHolder()`重新绑定数据，`mAttachedScrap`和`mCachedViews`没有发生变化，是直接使用的，不需要重新绑定数据，而`mRecyclerPool`中的ViewHolder的内容信息已经被抹除，需要重新绑定数据。所以在RecyclerView来回滚动时，`mCachedViews`缓存池的使用效率最高。

**总的来说：`RecyclerView`着重在两个场景缓存和回收的优化，一是：在数据更新时，使用`Scrap`进行局部更新，尽可能复用原来viewHolder，减少绑定数据的工作；二是：在滑动的时候，重复利用原来的ViewHolder，尽可能减少重复创建ViewHolder和绑定数据的工作。最终思想就是，能不创建就不创建，能不重新绑定就不重新绑定，尽可能减少重复不必要的工作。**