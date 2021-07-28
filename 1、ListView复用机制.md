ListView继承与AbsListView

AbsListView有复用机制，所以ListView有复用属性，AbsListView的复用能力全部来自一个内部类RecycleBin



* ```ActivityView```其实就是在UI屏幕上可见的视图，也是与用户进行交互的View
* ```ScrapView```，也就是废弃的View，已经无法与用户进行交互了，这样在UI视图改变的时候就没有绘制这些无用视图的必要了。他将会被```RecycleBin```存储到```mScrapView```数组当中，但是没有被销毁掉，目的是为了二次复用，也就是间接复用。当新的View需要显示的时候，先判断```mActivityView```中是否存在，如果存在那么我们就可以从```mActivityView```数组当中直接取出复用，也就是直接复用，否则的话从```mScrapView```数组当中进行判断，如果存在，那么二次复用当前的视图，如果不存在，那么就需要```inflate View```了。
* 第一次打开列表往上滑的时候所有的view都是在创建新的，下滑的时候会获取缓存里的View

### 如果顶部的这个View划出屏幕就开始进行回收

```java
 // 当顶部View滑出屏幕
View first = getChildAt(0);
while (first.getBottom() < listTop) {
	AbsListView.LayoutParams layoutParams = (LayoutParams) first.getLayoutParams();
	if (recycleBin.shouldRecycleViewType(layoutParams.viewType)) {
        //将View放到废料View区   first是当前这个View,mFirstPosition是屏幕显示的第一个子元素的位置下标
		recycleBin.addScrapView(first, mFirstPosition);
	}
	detachViewFromParent(first);
	first = getChildAt(0);
	mFirstPosition++;
}
```

### 回收处理

```java
/**
 * 进行回收这个View
 * <p>
 * 如果列表数据没有改变，或者适配器有稳定的id，视图
 * 的瞬态将被保留以供以后检索
 * 
 * @param 要回收的View
 * @param 视图在其父视图中的位置position
 */
void addScrapView(View scrap, int position) {
    final AbsListView.LayoutParams lp = (AbsListView.LayoutParams) scrap.getLayoutParams();
    if (lp == null) {
        // Can't recycle, but we don't know anything about the view.
        // Ignore it completely.
        return;
    }
    lp.scrappedFromPosition = position;
    //不移除除头布局与为布局且不想移除的View(当viewtype<0)  将这些不想移除的view添加到跳过回收View的List中
    final int viewType = lp.viewType;
    if (!shouldRecycleViewType(viewType)) {
        // Can't recycle. If it's not a header or footer, which have
        // special handling and should be ignored, then skip the scrap
        // heap and we'll fully detach the view later.
        if (viewType != ITEM_VIEW_TYPE_HEADER_OR_FOOTER) {
            getSkippedScrap().add(scrap);
        }
        return;
    }
    scrap.dispatchStartTemporaryDetach();
    // The the accessibility state of the view may change while temporary
    // detached and we do not allow detached views to fire accessibility
    // events. So we are announcing that the subtree changed giving a chance
    // to clients holding on to a view in this subtree to refresh it.
    notifyViewAccessibilityStateChangedIfNeeded(
            AccessibilityEvent.CONTENT_CHANGE_TYPE_SUBTREE);
    // Don't scrap views that have transient state.
    final boolean scrapHasTransientState = scrap.hasTransientState();
    if (scrapHasTransientState) {
        if (mAdapter != null && mAdapterHasStableIds) {
            // If the adapter has stable IDs, we can reuse the view for
            // the same data.
            if (mTransientStateViewsById == null) {
                mTransientStateViewsById = new LongSparseArray<>();
            }
            mTransientStateViewsById.put(lp.itemId, scrap);
        } else if (!mDataChanged) {
            // If the data hasn't changed, we can reuse the views at
            // their old positions.
            if (mTransientStateViews == null) {
                mTransientStateViews = new SparseArray<>();
            }
            mTransientStateViews.put(position, scrap);
        } else {
            // Otherwise, we'll have to remove the view and start over.
            clearScrapForRebind(scrap);
            getSkippedScrap().add(scrap);
        }
    } else {
        clearScrapForRebind(scrap);
        if (mViewTypeCount == 1) {
            //如果只有一种view类型那么将直接添加到当前回收站List中  
            mCurrentScrap.add(scrap);
        } else {
            //如果多种viewType那么就以viewtype为数组的下标，对应的List进行添加回收view
            mScrapViews[viewType].add(scrap);
        }
        if (mRecyclerListener != null) {
            mRecyclerListener.onMovedToScrapHeap(scrap);
        }
    }
}
```

### 获取回收的的View

```java
/**
 * 返回一个来自ScrapViews集合的视图。这些都是无序的
 */
View getScrapView(int position) {
    //获取到viewType
    final int whichScrap = mAdapter.getItemViewType(position);
    if (whichScrap < 0) {
        return null;
    }
    //viewtype的类型个数只有一种的话
    if (mViewTypeCount == 1) {
        return retrieveFromScrap(mCurrentScrap, position);
    } else if (whichScrap < mScrapViews.length) {
        //多种view类型的话  就另一种数组中取
        return retrieveFromScrap(mScrapViews[whichScrap], position);
    }
    return null;
}
```

```java
private View retrieveFromScrap(ArrayList<View> scrapViews, int position) {
    final int size = scrapViews.size();
    if (size > 0) {
        // See if we still have a view for this position or ID.
        // Traverse backwards to find the most recently used scrap view
        for (int i = size - 1; i >= 0; i--) {
            final View view = scrapViews.get(i);
            final AbsListView.LayoutParams params =
                    (AbsListView.LayoutParams) view.getLayoutParams();
            if (mAdapterHasStableIds) {
                final long id = mAdapter.getItemId(position);
                if (id == params.itemId) {
                    return scrapViews.remove(i);
                }
            } else if (params.scrappedFromPosition == position) {
                final View scrap = scrapViews.remove(i);
                clearScrapForRebind(scrap);
                return scrap;
            }
        }
        final View scrap = scrapViews.remove(size - 1);
        clearScrapForRebind(scrap);
        return scrap;
    } else {
        return null;
    }
}
```