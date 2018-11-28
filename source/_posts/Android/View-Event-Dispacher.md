---
title: View事件分发机制源码分析
date: 2017-01-05 13:07
tags:
categories: Android
description: View事件分发机制源码分析
---

# 1、事件传递规则概述
在解释事件分发机制之前，需要搞清楚几个概念。
1、事件:由于android设备对键盘依赖性的降低，导致触摸事件(MotionEvent)成为android最主要的事件，所以对于事件的分发，其实就是对MotionEvent对象的传递过程。
2、事件序列：从用户手指按下，到手里离开的这一系列事件的集合(可以看成用户在屏幕上的一个手势)。即事件序列以ACTION_DOWN开始，ACTION_UP结束：ACTION_DOWN-->ACTION_MOVE-->ACTION-->...->ACTION_UP事件。

---------
MotionEvent的分发主要涉及到以下几个方法：
```java
1、public boolean dispatchTouchEvent(MotionEvent ev);//当事件传递到当前View时调用。
2、public boolean onTouchEvent(MotionEvent event);//在dispatchTouchEvent中被间接调用。
3、public boolean onInterceptTouchEvent(MotionEvent ev);	//在dispatchTouchEvent中被调用，判断是否对MotionEvent进行拦截。
```
三个方法中最主要的是dispatchTouchEvent方法，其他两个方法只是在该方法中被调用，而该方法细节主要分两种情况：
## 1、当前View为ViewGroup
将会调用onInterceptTouchEvent()方法判断是否对事件进行拦截，如果拦截将会将事件交给当前控件的onTouchEvent()处理，否则将会分发给子控件处理(即调用子控件的dispatchTouchEvent方法)。
### 下面是ViewGroup消息处理流程的伪代码
```java
/**
 *这只是大概的流程，详细的源代码细节会在之后讲解
 */
boolean dispatchTouchEvent(MotionEvent e){
	boolean consumed = false;
	if(onInterceptTouchEvent(e)){
		consumed = onTouchEvent(e);
	}else{
		consume = childView.dispatchTouchEvent(e);
	}
	return consumed;
}
```
## 2、当前View为非ViewGroup的普通View
如果当前View设置了onTouchListener将会调用Listener对象中的onTouch方法，如果onTouch返回true直接返回，为false则调用当前View的onTouchEvent方法，默认View的onTouchEvent方法中会根据clickable属性或longClickable等相关属性判断是否调用onClickListener中的onClick方法。
### 下面是非ViewGroup的普通View分发事件流程的伪代码
```java
/**
 *这只是大概的流程，详细的源代码细节会在之后讲解
 */
boolean dispatchTouchEvent(MotionEvent e){
	boolean result = false;
	if(mOnTouchListener != null){
		result = mOnTouchListener.onTouch(this,e)；
	}
	if(result == false){
		result = onTouchEvent(e);
	}
}
boolean onTouchEvent(MotionEvent e){
	boolean result = false;
	switch(e.getAction()){
		case MotionEvent.ACTION_UP:
			if(mOnClickListener != null){
				mOnClickListener.onClick(this);
				result = true;
			}
			break;
		...
	}
	...
	return false;
}
```
一个点击事件产生后，它的传递过程主要按以下顺序：
分发过程：Activity->Window->ViewGroup->View；
按照这种过程一层一层分发下去，如果下层View的onTouchEvent方法没有处理则返回false，交由上一层处理，如果上一层也没有处理，再交由上一层，就是这样将未处理的事件返回给上一层处理。

# 源代码分析
源代码分析前，先将一些结论整理出来，带着结论看源码也许会更轻松，更容易理解，我们在源代码中对以下结论进行验证。
1、通常，一个**事件序列**只能被一个View拦截并消耗，因为一旦事件被拦截了，那么同一个事件序列内的所有事件都将直接交给该View处理，所以同一个事件序列中的事件不能同时由两个View处理，但是可以使用其他特殊方法做到，比如View将本该自己处理的事件通过调用其他View的onTouchEvent强行交给其他View处理。
2、某个View一旦决定拦截(onInterceptTouchEvent方法被调用并返回true)，那该事件序列中的所有事件都只能由它来处理，而且在该事件序列传递过程中这个View的onInterceptTouchEvent不会再被调用。换句话说，onInterceptTouchEvent一旦决定拦截会拦截一个事件序列。
3、某个View一旦开始处理事件，如果它不消耗ACTION_DOWN事件(即onTouchEvent返回了false)，那么同一个事件序列中的其他事件都不会在交给它来处理。
4、如果View不消耗除ACTION_DOWN以外的其他事件，那么这个点击事件会消失，此时父控件的onTouchEvent方法并不会被调用，并且当前View可以持续收到后续的事件，最终这些消失的点击事件会传递给Activity来处理。
5、ViewGroup的onInterceptTouchEvent方法默认返回false(默认不拦截任何事件)；View没有onInterceptTouchEvent方法，一旦事件传递给它，它按照onTouchEventListener.onTouch->onTouchEvent->onClickEventListener.onClick的顺序执行
6、一般都是先传递给父控件，再有父控件分发给子控件，通过requestDisallowInterceptTouchEvent方法可以在子控件干预父控件的事件分发过程，但是ACTION_DOWN事件除外。

---------
1、我们从**ViewGroup.dispatchTouchEvent**开始摘取重点进行分析
```java
public boolean dispatchTouchEvent(MotionEvent ev){
	...

	boolean handled = false;//设置标志位，判断是否已经处理了该事件

	//过滤掉本不应该发送到该控件的事件(比如该控件所在的窗口已经被遮挡了)
    if (onFilterTouchEventForSecurity(ev)) {
        final int action = ev.getAction();
        final int actionMasked = action & MotionEvent.ACTION_MASK;

        // 对事件序列的第一个事件ACTION_DOWN进行初始化处理
        if (actionMasked == MotionEvent.ACTION_DOWN) {
			//清除前一个事件序列的相关信息,这些信息保存在名为mFirstTouchTarget的TouchTarget对象中，
			//TouchTarget是一个链表，里面保存了应当处理该事件的View链
            cancelAndClearTouchTargets(ev);
            resetTouchState();
        }

        // 检查本控件是否需要拦截该事件
        final boolean intercepted;
        if (actionMasked == MotionEvent.ACTION_DOWN
                || mFirstTouchTarget != null) {
			//该事件为ACTION_DOWN，或者已经有子控件需要处理该事件序列

			//判断是否允许拦截，FLAG_DISALLOW_INTERCEPT标志位可通过requestDisallowInterceptTouchEvent方法设置，
			//子控件就可以通过该方法来干预父控件的事件分发过程，验证了结论6
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            if (!disallowIntercept) {
				//允许拦截该事件，则调用onInterceptTouchEvent方法查看是否需要拦截该事件，
				//intercepted记录下返回值，true为拦截，false为不拦截
                intercepted = onInterceptTouchEvent(ev);
                ev.setAction(action); //还原Action，防止onInterceptTouchEvent修改了Action
            } else {
				//不允许拦截，则直接设置为未拦截。
                intercepted = false;
            }
        } else {
			//不是事件序列的第一个事件，而且没有子控件“愿意”处理该事件序列。
            intercepted = true;
        }

        // If intercepted, start normal event dispatch. Also if there is already
        // a view that is handling the gesture, do normal event dispatch.
        if (intercepted || mFirstTouchTarget != null) {
            ev.setTargetAccessibilityFocus(false);
        }

        // Check for cancelation.
        final boolean canceled = resetCancelNextUpFlag(this)
                || actionMasked == MotionEvent.ACTION_CANCEL;

        // Update list of touch targets for pointer down, if needed.
        final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
        TouchTarget newTouchTarget = null;
        boolean alreadyDispatchedToNewTouchTarget = false;
        if (!canceled && !intercepted) {
			//如果事件序列没有被取消而且该控件也没有拦截该事件

            // If the event is targeting accessiiblity focus we give it to the
            // view that has accessibility focus and if it does not handle it
            // we clear the flag and dispatch the event to all children as usual.
            // We are looking up the accessibility focused host to avoid keeping
            // state since these events are very rare.
            View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                    ? findChildWithAccessibilityFocus() : null;

            if (actionMasked == MotionEvent.ACTION_DOWN
                    || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                final int actionIndex = ev.getActionIndex(); // always 0 for down
                final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                        : TouchTarget.ALL_POINTER_IDS;

                // Clean up earlier touch targets for this pointer id in case they
                // have become out of sync.
                removePointersFromTouchTargets(idBitsToAssign);

                final int childrenCount = mChildrenCount;
                if (newTouchTarget == null && childrenCount != 0) {
                    final float x = ev.getX(actionIndex);
                    final float y = ev.getY(actionIndex);
                    // Find a child that can receive the event.
                    // Scan children from front to back.
					//根据子控件在Z方向的大小，对子控件进行排序，上面的子控件应该先处理触摸事件
                    final ArrayList<View> preorderedList = buildOrderedChildList();
                    final boolean customOrder = preorderedList == null
                            && isChildrenDrawingOrderEnabled();
                    final View[] children = mChildren;
                    for (int i = childrenCount - 1; i >= 0; i--) {
						//遍历所有的子控件

                        final int childIndex = customOrder
                                ? getChildDrawingOrder(childrenCount, i) : i;
                        final View child = (preorderedList == null)
                                ? children[childIndex] : preorderedList.get(childIndex);

                        // If there is a view that has accessibility focus we want it
                        // to get the event first and if not handled we will perform a
                        // normal dispatch. We may do a double iteration but this is
                        // safer given the timeframe.
                        if (childWithAccessibilityFocus != null) {
                            if (childWithAccessibilityFocus != child) {
								//当前子控件没有获取焦点，则跳过该子控件
								//先获取焦点，然后才能触发点击事件
                                continue;
                            }
							//如果子控件已获取焦点，重新迭代，这部分我也搞不懂，注释说为了更安全
                            childWithAccessibilityFocus = null;
                            i = childrenCount - 1;
                        }


						//canViewReceivePointerEvents检测子控件是否能接收触摸事件
						//不可见的，正在执行动画的View不能接收触摸事件
						//isTransformedTouchPointInView检测触摸点是否在子控件中
                        if (!canViewReceivePointerEvents(child)
                                || !isTransformedTouchPointInView(x, y, child, null)) {
							//不满足条件的，跳过该子控件
                            ev.setTargetAccessibilityFocus(false);
                            continue;
                        }

                        newTouchTarget = getTouchTarget(child);
                        if (newTouchTarget != null) {
							//查看child是否加入到mFirstTouchTarget链表中
                            // Child is already receiving touch within its bounds.
                            // Give it the new pointer in addition to the ones it is handling.
                            newTouchTarget.pointerIdBits |= idBitsToAssign;
                            break;
                        }

                        resetCancelNextUpFlag(child);

						//dispatchTransformedTouchEvent，对事件进行一些转换，比如坐标转换成相对子控件边界的坐标
						//并将转换后的事件分发给该子控件，该函数后面还会重点解析
                        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                            // Child wants to receive touch within its bounds.
                            mLastTouchDownTime = ev.getDownTime();
                            if (preorderedList != null) {
                                // childIndex points into presorted list, find original index
								//如果childIndex是指向排序后的列表，则需要找到在mChildren中的原始索引
                                for (int j = 0; j < childrenCount; j++) {
                                    if (children[childIndex] == mChildren[j]) {
                                        mLastTouchDownIndex = j;
                                        break;
                                    }
                                }
                            } else {
                                mLastTouchDownIndex = childIndex;
                            }
                            mLastTouchDownX = ev.getX();
                            mLastTouchDownY = ev.getY();
							//将该子控件添加到mFirstTouchTarget链表的头部
                            newTouchTarget = addTouchTarget(child, idBitsToAssign);
                            alreadyDispatchedToNewTouchTarget = true;
                            break;
                        }

                        // The accessibility focus didn't handle the event, so clear
                        // the flag and do a normal dispatch to all children.
                        ev.setTargetAccessibilityFocus(false);
                    }
                    if (preorderedList != null) preorderedList.clear();
                }

                if (newTouchTarget == null && mFirstTouchTarget != null) {
                    // Did not find a child to receive the event.
                    // Assign the pointer to the least recently added target.
					//没有子控件接收该事件
                    newTouchTarget = mFirstTouchTarget;
                    while (newTouchTarget.next != null) {
                        newTouchTarget = newTouchTarget.next;
                    }
                    newTouchTarget.pointerIdBits |= idBitsToAssign;
                }
            }
        }

        // Dispatch to touch targets.
		// 将事件分发给mFirstTouchTarget保存的事件处理View
        if (mFirstTouchTarget == null) {
            // No touch targets so treat this as an ordinary view.
			// 没有子控件接收该事件，则分发给自己，即调用自己的onTouchEvent
            handled = dispatchTransformedTouchEvent(ev, canceled, null,
                    TouchTarget.ALL_POINTER_IDS);
        } else {
            // Dispatch to touch targets, excluding the new touch target if we already
            // dispatched to it.  Cancel touch targets if necessary.
            TouchTarget predecessor = null;
            TouchTarget target = mFirstTouchTarget;
            while (target != null) {
				//遍历TouchTarget对象，将事件分发给所有需要处理该事件的所有控件
                final TouchTarget next = target.next;
                if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
					//如果在之前的处理过程中已经发给了target中保存的View，则直接设置handle为true，表示已经处理过了该事件
                    handled = true;
                } else {
                    final boolean cancelChild = resetCancelNextUpFlag(target.child)
                            || intercepted;
					//转换坐标后将事件分发给子控件
                    if (dispatchTransformedTouchEvent(ev, cancelChild,
                            target.child, target.pointerIdBits)) {
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
                        continue;
                    }
                }
                predecessor = target;
                target = next;
            }
        }
		...
    }
	...
	return handled;
}
```
2、接下来分析**ViewGroup.dispatchTransformedTouchEvent**方法,我们摘取里面最重要的一部分进行解析
```java
	private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel, View child, int desiredPointerIdBits) {

        ...
        // Perform any necessary transformations and dispatch.
        if (child == null) {
			//调用父类的dispatchTouchEvent方法，也就是调用View的dispatchEvent
			//结论5说过，View接收到事件，按照onTouchListener.onTouch->onTouchEvent->onClickListener.onClick顺序执行，
			//onTouchListener默认为空，所以这句话相当于调用自己的onTouchEvent方法
			//关于View.diapatchTouchEvent方法，后面还会解析
            handled = super.dispatchTouchEvent(transformedEvent);
        } else {
			//计算相对于子控件边界的坐标值
            final float offsetX = mScrollX - child.mLeft;
            final float offsetY = mScrollY - child.mTop;
            transformedEvent.offsetLocation(offsetX, offsetY);
            if (! child.hasIdentityMatrix()) {
                transformedEvent.transform(child.getInverseMatrix());
            }

			//将转换后的事件对象分发给子控件
            handled = child.dispatchTouchEvent(transformedEvent);
        }
		...
		return handled;
    }
```
3、现在我们对**View.dispatchTouchEvent**进行解析
```java

   	public boolean dispatchTouchEvent(MotionEvent event) {

		...
        if (onFilterTouchEventForSecurity(event)) {
            //noinspection SimplifiableIfStatement
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
				//如果该控件设置了OnTouchEventListener监听器，则交由监听器处理
                result = true;
            }

            if (!result && onTouchEvent(event)) {
				//如果未设置OnTouchEventListener监听器或者监听器返回false，则调用自己的onTouchEvent方法
                result = true;
            }
        }
		...
        return result;
    }
```
4、最后我们看一下控件默认的**View.onTouchEvent**方法
```java

	public boolean onTouchEvent(MotionEvent event) {
        final float x = event.getX();
        final float y = event.getY();
        final int viewFlags = mViewFlags;
        final int action = event.getAction();

        if ((viewFlags & ENABLED_MASK) == DISABLED) {
            if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
				//ACTION_UP触发修改PFLAG_PRESSED标志位，
				//同时更换drawableStateList设置的不同状态下的drawable资源
                setPressed(false);
            }
            // A disabled view that is clickable still consumes the touch
            // events, it just doesn't respond to them.
			// 即使DISABLED状态下的控件，只要满足：
			// CLICKABLE(可点击)，
			// LONG_CLICKABLE(可长按)，
			// CONTEXT_CLICKABLE(手写笔、鼠标相关，基本不用考虑)
			// 仍然消耗MotionEvent事件，只是不做响应
            return (((viewFlags & CLICKABLE) == CLICKABLE
                    || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
                    || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE);
        }

        if (mTouchDelegate != null) {
			//委托的触摸代理，将本控件一个区域映射到另一个控件的某个区域
			//如果点击了委托的区域则将MotionEvent交给委托的View控件来处理
			//详细的内容可以自行阅读TouchDelegate源码
            if (mTouchDelegate.onTouchEvent(event)) {
                return true;
            }
        }

        if (((viewFlags & CLICKABLE) == CLICKABLE ||
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
                (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {
            switch (action) {
                case MotionEvent.ACTION_UP:
                    boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                        // take focus if we don't have it already and we should in
                        // touch mode.
						// 如果没有获得焦点，那么点击后让其获取焦点
                        boolean focusTaken = false;
                        if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                            focusTaken = requestFocus();
                        }

                        if (prepressed) {
                            // The button is being released before we actually
                            // showed it as pressed.  Make it show the pressed
                            // state now (before scheduling the click) to ensure
                            // the user sees it.
                            setPressed(true, x, y);
                       }

                        if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                            // This is a tap, so remove the longpress check
                            removeLongPressCallback();

                            // Only perform take click actions if we were in the pressed state
                            if (!focusTaken) {
                                // Use a Runnable and post this rather than calling
                                // performClick directly. This lets other visual state
                                // of the view update before click actions start.
                                if (mPerformClick == null) {
                                    mPerformClick = new PerformClick();
                                }
                                if (!post(mPerformClick)) {
									//performClick中将会执行已经设置了onClickListener.onClick方法
                                    performClick();
                                }
                            }
                        }

                        if (mUnsetPressedState == null) {
                            mUnsetPressedState = new UnsetPressedState();
                        }

                        if (prepressed) {
                            postDelayed(mUnsetPressedState,
                                    ViewConfiguration.getPressedStateDuration());
                        } else if (!post(mUnsetPressedState)) {
                            // If the post failed, unpress right now
                            mUnsetPressedState.run();
                        }

                        removeTapCallback();
                    }
                    mIgnoreNextUpEvent = false;
                    break;

                case MotionEvent.ACTION_DOWN:
					mHasPerformedLongPress = false;

                    if (performButtonActionOnTouchDown(event)) {
                        break;
                    }

                    // Walk up the hierarchy to determine if we're inside a scrolling container.
                    boolean isInScrollingContainer = isInScrollingContainer();

                    // For views inside a scrolling container, delay the pressed feedback for
                    // a short period in case this is a scroll.
                    if (isInScrollingContainer) {
                        mPrivateFlags |= PFLAG_PREPRESSED;
                        if (mPendingCheckForTap == null) {
                            mPendingCheckForTap = new CheckForTap();
							//CheckForTap的run方法中会调用checkForLongClick
                        }
                        mPendingCheckForTap.x = event.getX();
                        mPendingCheckForTap.y = event.getY();
                        postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
                    } else {
                        // Not inside a scrolling container, so show the feedback right away
                        setPressed(true, x, y);
						//该方法中会执行一个延时的LongClick
                        checkForLongClick(0);
                    }
					break;
                case MotionEvent.ACTION_CANCEL:
					...
                    break;

                case MotionEvent.ACTION_MOVE:
                    ...
                    break;
            }

            return true;
        }

        return false;
    }
```

到此已经大致的将View的事件分发机制讲解完了。