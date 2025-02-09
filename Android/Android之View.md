## View

### View的绘制流程

#### setContentView

##### DecorView的创建

![[DecorView的创建流程.png]]

1. `ActivityThread#performLaunchActivity`：当`Activity`启动时，在`ActivityThread `会通过`Instrumentation `创建`Activity`调用其`onCreate`方法，并创建`PhoneWindow`实例保存在`Activity`中；
2. `PhoneWindow#setContentView` ：`Activity`的`setContentView` 由`PhoneWindow`完成；
   1. `generateDecor`：创建一个`DecorView`；
   2. `onResourcesLoaded`：加载`DecorView`布局，默认为`R.layout.screen_simple`；
   3. `generateLayout`：根据`id（ID_ANDROID_CONTENT）`初始化根布局`mContentParent`；
   4. 通过`mLayoutInflater.inflate(layoutResID, mContentParent)`加载`Activity`布局；

##### AppCompatActivity

androidx之后Activity默认继承自AppCompatActivity，hook原生的setContentView流程，添加了一些方法；
![[View的绘制流程.png]]

1. createSubDecor：将原生DecorView content 的 View 复制到 R.id.action_bar_activity_content，并将content替换；

#### 草稿

1. performMeasure 最多执行三次？
2. UI线程不是只能在主线程刷新？
   - 哪个线程创建的ViewRootImpl

### DecorView添加到Window

![[DecorView添加到Window的流程.png]]

### LayoutInflater

#### inflate

``` java
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {  
    synchronized (mConstructorArgs) {  
	      ...
        try {  
            advanceToRootNode(parser);   
  
            if (TAG_MERGE.equals(name)) {  //1
                if (root == null || !attachToRoot) {  
                    throw new InflateException("<merge /> can be used only with a valid "  
                            + "ViewGroup root and attachToRoot=true");  
                }  
  
                rInflate(parser, root, inflaterContext, attrs, false);  
            } else {  
             
                final View temp = createViewFromTag(root, name, inflaterContext, attrs);  //2
  
                ViewGroup.LayoutParams params = null;  
  
                if (root != null) {  //4
							params = root.generateLayoutParams(attrs);  
                    if (!attachToRoot) {  
                        temp.setLayoutParams(params);  
                    }  
                }  
                
                // Inflate all children under temp against its context.  
                rInflateChildren(parser, temp, attrs, true);  // 3

				    if (root != null && attachToRoot) {  
                    root.addView(temp, params);  //4
                }  
  
					 if (root == null || !attachToRoot) {  
                    result = temp;  
                }  
		      }  
  
        } catch (XmlPullParserException e) {  
				...
        } catch (Exception e) {  
				...
        } finally {  
			...
        }  
  
        return result;  
    }  
}
```

1. `merge`标签  通过减少视图节点来优化布局，`root`不能为空，而且`attachToRoot`必须为`true`，如果与`Activity`也是用`FrameLayout`，可以替换成`merge`，这样减少一层布局;
2. 创建当前标签对应`View`；
3. 遍历创建`View`所有子`View`。
4. `root != null && !attachToRoot` 只使用布局参数，不让其处于同一容器中；
5. `root != null && attachToRoot` 将创建的`View`直接添加到父容器中；
6. `root == null || !attachToRoot`不使用布局参数，

#### createViewFromTag

``` java
View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,  
        boolean ignoreThemeAttr) {  
...
View view = tryCreateView(parent, name, context, attrs);  
  
if (view == null) {  
    final Object lastContext = mConstructorArgs[0];  
    mConstructorArgs[0] = context;  
    try {  
        if (-1 == name.indexOf('.')) {  //1
            view = onCreateView(context, parent, name, attrs);  
        } else {  
            view = createView(context, name, null, attrs);  
        }  
    } finally {  
        mConstructorArgs[0] = lastContext;  
    }  
}  
...  
return view;
}
public final View createView(@NonNull Context viewContext, @NonNull String name,  
                             @Nullable String prefix, @Nullable AttributeSet attrs)  
        throws ClassNotFoundException, InflateException {  
      ...
    Constructor<? extends View> constructor = sConstructorMap.get(name);  
	  ...
    Class<? extends View> clazz = null;  
  
    if (constructor == null) {  
        // Class not found in the cache, see if it's real, and try to add it  
        clazz = Class.forName(prefix != null ? (prefix + name) : name, false,  
                mContext.getClassLoader()).asSubclass(View.class);  //2
  
        if (mFilter != null && clazz != null) {  
            boolean allowed = mFilter.onLoadClass(clazz);  
            if (!allowed) {  
                failNotAllowed(name, prefix, viewContext, attrs);  
            }  
        }  
        constructor = clazz.getConstructor(mConstructorSignature); //3 
        constructor.setAccessible(true);  
        sConstructorMap.put(name, constructor);  
    } else {  
        // If we have a filter, apply it to cached constructor  
        if (mFilter != null) {  
            // Have we seen this name before?  
            Boolean allowedState = mFilterMap.get(name);  
            if (allowedState == null) {  
                // New class -- remember whether it is allowed  
                clazz = Class.forName(prefix != null ? (prefix + name) : name, false,  
                        mContext.getClassLoader()).asSubclass(View.class);  
  
                boolean allowed = clazz != null && mFilter.onLoadClass(clazz);  
                mFilterMap.put(name, allowed);  
                if (!allowed) {  
                    failNotAllowed(name, prefix, viewContext, attrs);  
                }  
            } else if (allowedState.equals(Boolean.FALSE)) {  
                failNotAllowed(name, prefix, viewContext, attrs);  
            }  
        }  
    }  
  
    Object lastContext = mConstructorArgs[0];  
    mConstructorArgs[0] = viewContext;  
    Object[] args = mConstructorArgs;  
    args[1] = attrs;  
  
    try {  
        final View view = constructor.newInstance(args);  //3
        if (view instanceof ViewStub) {  //4
         
            final ViewStub viewStub = (ViewStub) view;  
            viewStub.setLayoutInflater(cloneInContext((Context) args[0]));  
        }  
        return view;  
    } finally {  
        mConstructorArgs[0] = lastContext;  
    }  
}
```

1. `SDK View`会默认添加`android.view.`前缀后与自定义`View`执行相同逻辑；
2. 通过`Class.forName`加载对应`View`的`Class`;
3. 反射调用View的2个参数的构造方法`View(Context context,AttributeSet attrs)`
4. ViewStub 懒加载View，默认不显示，在需要时，通过inflate方法加载属性中保持的布局。

#### rInflateChildren

``` java
final void rInflateChildren(XmlPullParser parser, View parent, AttributeSet attrs,  
        boolean finishInflate) throws XmlPullParserException, IOException {  
    rInflate(parser, parent, parent.getContext(), attrs, finishInflate);  
}
void rInflate(XmlPullParser parser, View parent, Context context,  
        AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {  
  
    final int depth = parser.getDepth();  
    int type;  
    boolean pendingRequestFocus = false;  
  
    while (((type = parser.next()) != XmlPullParser.END_TAG ||  
            parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {  
  
        if (type != XmlPullParser.START_TAG) {  
            continue;  
        }  
  
        final String name = parser.getName();  
  
        if (TAG_REQUEST_FOCUS.equals(name)) {  
            pendingRequestFocus = true;  
            consumeChildElements(parser);  
        } else if (TAG_TAG.equals(name)) {  
            parseViewTag(parser, parent, attrs);  
        } else if (TAG_INCLUDE.equals(name)) {  
            if (parser.getDepth() == 0) {  //1
                throw new InflateException("<include /> cannot be the root element");  
            }  
            parseInclude(parser, context, parent, attrs);  
        } else if (TAG_MERGE.equals(name)) {  //2
            throw new InflateException("<merge /> must be the root element");  
        } else {  
            final View view = createViewFromTag(parent, name, context, attrs);  
            final ViewGroup viewGroup = (ViewGroup) parent;  
            final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);  
            rInflateChildren(parser, view, attrs, true);  
            viewGroup.addView(view, params);  
        }  
    }  
  
    if (pendingRequestFocus) {  
        parent.restoreDefaultFocus();  
    }  
  
    if (finishInflate) {  
        parent.onFinishInflate();  
    }  
}
```

1. `include`标签不能作为根节点, include标签中的`layout_*`属性和`id`会替换掉`include`视图根节点的对应属性和id；
2. `merge`必须放在布局文件的根节点上；

### 事件分发机制

#### 草稿?

1. 事件分发的Activity的流程
2. 手指移出View view的事件的？
3. 长按触发时间不同的版本不同？
4. 事件分发的责任链模式？
5. 单指还是多指 down事件重置状态都指执行一次
6. 倒序遍历寻找处理的子View
7. 没有子View处理 自己处理？
8. Down事件不能拦截 子View都拿不到事件怎么设置 FLAG_DISALLOW_INTERCEPT
9. ViewPager与ListView的滑动冲突解决？

#### ViewGroup.dispatchTouchEvent

1. DOWN事件，重置 FLAG_DISALLOW_INTERCEPT，遍历链表mFirstTouchTarget，发送cancel事件到子View，并清空链表。
2. 针对DOWN事件或者已有子View处理了事件的情况判断是否拦截，需要同时满足2个条件
   1. 子View未设置FLAG_DISALLOW_INTERCEPT不允许拦截；
   2. onInterceptTouchEvent返回true；
3. 未拦截时 ，分发DOWN事件，按照Z轴顺序遍历所有子View，如果dispatchTransformedTouchEvent返回true， 将子view封装成TouchTarget保存在mFirstTouchTarget中；
4. 如果没有子View处理，则交给自己处理。
5. 除DOWN事件外，其他事件则不在遍历所有子View，只在mFirstTouchTarget 查找对应的子View处理；

```java
public boolean dispatchTouchEvent(MotionEvent ev) {  
//        ......  
        boolean handled = false;  
  
        final int action = ev.getAction();  
        final int actionMasked = action & MotionEvent.ACTION_MASK;  
        //1  
        if (actionMasked == MotionEvent.ACTION_DOWN) { //1  
            cancelAndClearTouchTargets(ev);  
            resetTouchState();  
        }  
        //2  
        final boolean intercepted;  
        if (actionMasked == MotionEvent.ACTION_DOWN //2  
                || mFirstTouchTarget != null) {  
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;  
            if (!disallowIntercept) {  
                intercepted = onInterceptTouchEvent(ev);  
                ev.setAction(action);   
} else {  
                intercepted = false;  
            }  
        } else {  
            intercepted = true;  
        }  
//        ......  
        final boolean canceled = resetCancelNextUpFlag(this)  
                || actionMasked == MotionEvent.ACTION_CANCEL;  
        final boolean isMouseEvent = ev.getSource() == InputDevice.SOURCE_MOUSE;  
        final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0  
                && !isMouseEvent;  
        TouchTarget newTouchTarget = null;  
        boolean alreadyDispatchedToNewTouchTarget = false;  
        if (!canceled && !intercepted) {  
            //......  
            if (actionMasked == MotionEvent.ACTION_DOWN  //3
                    || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)  
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {  
                final int actionIndex = ev.getActionIndex(); // always 0 for down  
                final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)  
                        : TouchTarget.ALL_POINTER_IDS;  
                removePointersFromTouchTargets(idBitsToAssign);  
                final int childrenCount = mChildrenCount;  
                if (newTouchTarget == null && childrenCount != 0) {  
                    final float x =  
                            isMouseEvent ? ev.getXCursorPosition() : ev.getX(actionIndex);  
                    final float y =  
                            isMouseEvent ? ev.getYCursorPosition() : ev.getY(actionIndex);  
                    final ArrayList<View> preorderedList = buildTouchDispatchChildList();  
                    final boolean customOrder = preorderedList == null  
                            && isChildrenDrawingOrderEnabled();  
                    final View[] children = mChildren;  
                    for (int i = childrenCount - 1; i >= 0; i--) {  //3.1
                        final int childIndex = getAndVerifyPreorderedIndex(  
                                childrenCount, i, customOrder);  
                        final View child = getAndVerifyPreorderedView(  
                                preorderedList, children, childIndex);  
                        //......  
                        if (!child.canReceivePointerEvents()  
                                || !isTransformedTouchPointInView(x, y, child, null)) {  
                            ev.setTargetAccessibilityFocus(false);  
                            continue;                        }  
  
                        newTouchTarget = getTouchTarget(child);  
                        if (newTouchTarget != null) {  
                            newTouchTarget.pointerIdBits |= idBitsToAssign;  
                            break;                        }  
  
                        resetCancelNextUpFlag(child);  
                        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) { // 3.2
                            mLastTouchDownTime = ev.getDownTime();  
                            if (preorderedList != null) {  
                                for (int j = 0; j < childrenCount; j++) {  
                                    if (children[childIndex] == mChildren[j]) {  
                                        mLastTouchDownIndex = j;  
                                        break;                                    }  
                                }  
                            } else {  
                                mLastTouchDownIndex = childIndex;  
                            }  
                            mLastTouchDownX = ev.getX();  
                            mLastTouchDownY = ev.getY();  
                            newTouchTarget = addTouchTarget(child, idBitsToAssign);  //3.3
                            alreadyDispatchedToNewTouchTarget = true;  
                            break;                        }  
                        ev.setTargetAccessibilityFocus(false);  
                    }  
                    if (preorderedList != null) preorderedList.clear();  
                }  
  
                if (newTouchTarget == null && mFirstTouchTarget != null) {  
  
                    newTouchTarget = mFirstTouchTarget;  
                    while (newTouchTarget.next != null) {  
                        newTouchTarget = newTouchTarget.next;  
                    }  
                    newTouchTarget.pointerIdBits |= idBitsToAssign;  
                }  
            }  
        }  
        if (mFirstTouchTarget == null) {  //4
            handled = dispatchTransformedTouchEvent(ev, canceled, null,  
                    TouchTarget.ALL_POINTER_IDS);  
        } else {  
            TouchTarget predecessor = null;  
            TouchTarget target = mFirstTouchTarget;  
            while (target != null) {  
                final TouchTarget next = target.next;  
                if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {  
                    handled = true;  
                } else {  
                    final boolean cancelChild = resetCancelNextUpFlag(target.child)  
                            || intercepted;  
                    if (dispatchTransformedTouchEvent(ev, cancelChild,  
                            target.child, target.pointerIdBits)) {  //5
                        handled = true;  
                    }  
                    if (cancelChild) {  
                        if (predecessor == null) {  
                            mFirstTouchTarget = next;  
                        } else {  
                            predecessor.next = next;  
                        }  
                        target.recycle();  
                        target = next;  
                        continue;                    }  
                }  
                predecessor = target;  
                target = next;  
            }  
        }  
//        ......  
        return handled;  
    }
```

总结：

1. mFirstTouchTarget 为链表 数据结构，针对多指触摸，每个结点分别对应一个View；

### PhotoView

1. onSizeChange 调用时机
2. GestureDetectorListener
3. 属性动画？

### RecyclerView

#### 缓存与复用

#### 四级缓存

1. `mChangedScrap`和`mAttachedScrap` 用来缓存移除屏幕内的`ViewHolder`;
2. `mCachedViews` 用来缓存移除屏幕之外的`ViewHolder`;
3. `mViewCacheExtension` 这个创建和缓存完全由开发者自己控制，系统未往这里添加数据;
4. `RecycledViewPool` `ViewHolder`缓存池。
