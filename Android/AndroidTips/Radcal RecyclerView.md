# Components
- RecyclerView
> Creates, binds ,recycles.

- Adapter
> Provides data

- LayoutManager
> Lays out & positions. Linear -/ GridLayoutManager

- ItemDecoration
> Adds offsets, draws over / under

- ItemAimator
> Animates changes. DefaultItemAnimator

- ItemTouchHelper
> Handles drag&drop,swipe-to-dismiss. SimpleItemTouchHelper / SimpleCallback

- SnapHelper
> Creates ViewPager-like scrolls & flings. LinearSnapHelper

- DiffUtil
> Calculates changes for you.

# GridLayoutManager
- setSpanCount(int spanCount) and setSpanSizeLookUp()
> Suppose you want 1,2,3,4 columns on different item type, then you should setSpanCount with value 12 (as 1x2x3x4), and handles item span size on SpanSizeLookup
