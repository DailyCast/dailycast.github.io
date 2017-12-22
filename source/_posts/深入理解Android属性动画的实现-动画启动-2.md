---
title: 深入理解Android属性动画的实现(动画启动)-2
date: 2017-12-21 11:46:13
tags:
	- Android
	- 动画
category: Android
comments: true
---
### 属性动画的启动分析
在本文中,我们会分析属性动画如何启动的而且和Andoid `黄油计划`有什么关系

我们看看 `ObjectAnimator.start()`

```java
public void start() {
        AnimationHandler.getInstance().autoCancelBasedOn(this);
        if (DBG) {
            Log.d(LOG_TAG, "Anim target, duration: " + getTarget() + ", " + getDuration());
            for (int i = 0; i < mValues.length; ++i) {
                PropertyValuesHolder pvh = mValues[i];
                Log.d(LOG_TAG, "   Values[" + i + "]: " +
                    pvh.getPropertyName() + ", " + pvh.mKeyframes.getValue(0) + ", " +
                    pvh.mKeyframes.getValue(1));
            }
        }
        super.start();
    }
```
`AnimationHandler.getInstance().autoCancelBasedOn(this)` cancel 相同Target和相同属性的动画  
AnimationHandler 实例在线程局部单例。`autoCancelBasedOn(this)`会遍历 `AnimationHandler `实例持有的所有未完成的 `ValueAnimator`实例，cancel 掉符合条件的动画。

紧接着 `super.start()` 调用了`ValueAnimator.start()`  

```java
@Override
 public void start() {
     start(false);
 }
```
调用了带参数的 `start`方法

```java
private void start(boolean playBackwards) {
        if (Looper.myLooper() == null) {
            throw new AndroidRuntimeException("Animators may only be run on Looper threads");
        }
        mReversing = playBackwards;
        
			....
			
        mStarted = true;
        mPaused = false;
        mRunning = false;
        mAnimationEndRequested = false;
        // Resets mLastFrameTime when start() is called, so that if the animation was running,
        // calling start() would put the animation in the
        // started-but-not-yet-reached-the-first-frame phase.
        mLastFrameTime = 0;
        AnimationHandler animationHandler = AnimationHandler.getInstance();
        animationHandler.addAnimationFrameCallback(this, (long) (mStartDelay * sDurationScale));

 		....
    }
```

我们略去与主流程无关的逻辑代码。这个方法标记了动画的状态，如

```java
 mStarted = true;
 mPaused = false;
 mRunning = false;
```

然后这个方法结束啦，有没有很疑惑？有没有很懵逼？动画怎么执行的？什么时候执行的？  
要解答这问题，我们还是要分析 这两行不起眼的代码

```java
AnimationHandler animationHandler = AnimationHandler.getInstance();
animationHandler.addAnimationFrameCallback(this, (long) (mStartDelay * sDurationScale));
```
第一行在当前线程获取 `AnimationHandler`实例；  
第二行注册了一个callback。

```java
/**
     * Register to get a callback on the next frame after the delay.
     */
    public void addAnimationFrameCallback(final AnimationFrameCallback callback, long delay) {
        if (mAnimationCallbacks.size() == 0) {
            getProvider().postFrameCallback(mFrameCallback);
        }
        if (!mAnimationCallbacks.contains(callback)) {
            mAnimationCallbacks.add(callback);
        }

        if (delay > 0) {
            mDelayedCallbackStartTime.put(callback, (SystemClock.uptimeMillis() + delay));
        }
    }
```  
***注释说明:callback 会在delay 时间后的下一个 frame 获得回调。***

```java
if (mAnimationCallbacks.size() == 0) {
   getProvider().postFrameCallback(mFrameCallback);
}
```
这两行代码 做了一件事情:保证一个`ValueAnimator`对象只向Provider注册一次 `mFrameCallback`

瞅瞅 `getProvider`是啥？

```java
private AnimationFrameCallbackProvider getProvider() {
    if (mProvider == null) {
        mProvider = new MyFrameCallbackProvider();
    }
    return mProvider;
    }
```
创建了一个 `MyFrameCallbackProvider`实例， `MyFrameCallbackProvider` 继承  `AnimationFrameCallbackProvider`

```java
public interface AnimationFrameCallbackProvider {
    void postFrameCallback(Choreographer.FrameCallback callback);
    void postCommitCallback(Runnable runnable);
    long getFrameTime();
    long getFrameDelay();
    void setFrameDelay(long delay);
}
```
`AnimationFrameCallbackProvider`接口定义了一些回调接口，按照注释说明主要作用是 提高 `ValueAnimator`的可测性，通过这个接口隔离，我们可以自定义 定时脉冲，而不用使用系统默认的 `Choreographer`,这样我们可以在测试中使用任意的时间间隔的定时脉冲.既然可以方便测试，那肯定有API来更改Provider 吧？

果不其然，我们猜对了。

```java
public void setProvider(AnimationFrameCallbackProvider provider) {
    if (provider == null) {
       mProvider = new MyFrameCallbackProvider();
    } else {
       mProvider = provider;
    }
}
```

`ValueAnimator`提供了一个 `setProvider` 通过自定义的Provider 提供我们想要的任意时间间隔的回调，来更新动画。

明白了 接口`AnimationFrameCallbackProvider`的作用，也知道了一个新的名词`Choreographer`,它就是 ***Android 黄油计划*** 的核心。使用vsync（垂直同步）来协调View的绘制和动画的执行间隔。关于 `Choreographer` 在文章最后会做进一步解释    
我们知道了默认情况下系统使用`Choreographer`,我们可以简单的认为 它是一个与 绘制和动画有关的 _**消息处理器**_ 。  
继续我们的 代码 `AnimationHandler.addAnimationFrameCallback `

```java
public void addAnimationFrameCallback(final AnimationFrameCallback callback, long delay) {
   if (mAnimationCallbacks.size() == 0) {
       getProvider().postFrameCallback(mFrameCallback);
   }
   if (!mAnimationCallbacks.contains(callback)) {
       mAnimationCallbacks.add(callback);
   }

   if (delay > 0) {
       mDelayedCallbackStartTime.put(callback, (SystemClock.uptimeMillis() + delay));
   }
}
```

执行了 `getProvider().postFrameCallback(mFrameCallback)` 通过上面的分析我们知道 `getProvider()` 得到的是一个`MyFrameCallbackProvider`

```java
	/**
     * Default provider of timing pulse that uses Choreographer for frame callbacks.
     */
    private class MyFrameCallbackProvider implements AnimationFrameCallbackProvider {

        final Choreographer mChoreographer = Choreographer.getInstance();

        @Override
        public void postFrameCallback(Choreographer.FrameCallback callback) {
            mChoreographer.postFrameCallback(callback);
        }

        @Override
        public void postCommitCallback(Runnable runnable) {
            mChoreographer.postCallback(Choreographer.CALLBACK_COMMIT, runnable, null);
        }

        @Override
        public long getFrameTime() {
            return mChoreographer.getFrameTime();
        }

        @Override
        public long getFrameDelay() {
            return Choreographer.getFrameDelay();
        }

        @Override
        public void setFrameDelay(long delay) {
            Choreographer.setFrameDelay(delay);
        }
    }
```

注释说明系统默认使用的 `Choreographer`做定时脉冲来协调 frame 的更新 。 

`MyFrameCallbackProvider.postFrameCallback()`方法调用了` mChoreographer.postFrameCallback(callback)` 这里说明一下 Choreographer 实例也是线程局部单例的。从这些信息中我们知道了动画可以在子线程中执行的(注意:这不意味着可以在子线程更新UI)，但是这个子线程必须有Looper。 
接着分析 `Choreographer.postFrameCallback`方法

```java
public void postFrameCallback(FrameCallback callback) {
    postFrameCallbackDelayed(callback, 0);
}
    
public void postFrameCallbackDelayed(FrameCallback callback, long delayMillis) {
    if (callback == null) {
        throw new IllegalArgumentException("callback must not be null");
    }

    postCallbackDelayedInternal(CALLBACK_ANIMATION,
                callback, FRAME_CALLBACK_TOKEN, delayMillis);
} 

private void postCallbackDelayedInternal(int callbackType,
            Object action, Object token, long delayMillis) {
   ....
   synchronized (mLock) {
      final long now = SystemClock.uptimeMillis();
      final long dueTime = now + delayMillis;
      mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);
      if (dueTime <= now) {
          scheduleFrameLocked(now);
      } else {
          Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
          msg.arg1 = callbackType;
          msg.setAsynchronous(true);
          mHandler.sendMessageAtTime(msg, dueTime);
      }
   }
}
``` 
动画使用的 `callbackType`为 `CALLBACK_ANIMATION`，而默认支持 三种类型：

 `CALLBACK_INPUT`:处理输出相关的回调，最先被处理  
 `CALLBACK_ANIMATION`:处理动画相关的回调，在 `CALLBACK_TRAVERSAL`类型之前处理  
 `CALLBACK_TRAVERSAL`: 处理View 的layout 和 draw。该类型在所有其他异步消息处理完后开始处理，也基于这一点保证了界面不卡顿。
 
 每一个类型的 `callbackType` 都拥有一个`CallbackQueue` 我们的动画callback 会通过 `mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);` 保存在 `CallbackQueue`中成为一个链表中的一个节点。  
 处理完 callback 之后 下面会执行`scheduleFrameLocked()`  
 
 ```java
 private void scheduleFrameLocked(long now) {
    if (!mFrameScheduled) {
        mFrameScheduled = true;
        if (USE_VSYNC) {
            ...
            if (isRunningOnLooperThreadLocked()) {
                scheduleVsyncLocked();
            } else {
                Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
                msg.setAsynchronous(true);
                mHandler.sendMessageAtFrontOfQueue(msg);
            }
      } else {
          final long nextFrameTime = Math.max(
                        mLastFrameTimeNanos / TimeUtils.NANOS_PER_MS + sFrameDelay, now);
          Message msg = mHandler.obtainMessage(MSG_DO_FRAME);
          msg.setAsynchronous(true);
          mHandler.sendMessageAtTime(msg, nextFrameTime);
      }
   }
}
 ```
 默认情况下`USE_VSYNC`为true，当然设备厂商也可以设置为 false。如果 为false 会以 10 ms 为间隔计算下一次 doFrame 的时间，然后使用Handler来处理。  
 
 我们看看 ` scheduleVsyncLocked()` 这行代码做了何事?  
 
 ```java
 private void scheduleVsyncLocked() {
    mDisplayEventReceiver.scheduleVsync();
 }
 ```
 `mDisplayEventReceiver` 当`USE_VSYNC `为true 时 实例化成 `FrameDisplayEventReceiver`对象 ，它主要是和 native层 做交互，协调 vsync 信号。  
 `mDisplayEventReceiver.scheduleVsync()` 请求当下一帧开始时同步vsync信号。
 
 当vsync 信号来时 回调 JNI 层会回调`DisplayEventReceiver.dispatchVsync`方法 
 
 ```java
 // Called from native code.
 @SuppressWarnings("unused")
 private void dispatchVsync(long timestampNanos, int builtInDisplayId, int frame) {
     onVsync(timestampNanos, builtInDisplayId, frame);
    }
  
 public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
 }
 ```
 
 `FrameDisplayEventReceiver`实现了`doVsync`方法
 
 ```java
 private final class FrameDisplayEventReceiver extends DisplayEventReceiver
            implements Runnable {
        private boolean mHavePendingVsync;
        private long mTimestampNanos;
        private int mFrame;

        public FrameDisplayEventReceiver(Looper looper) {
            super(looper);
        }

        @Override
        public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
            ....
            
            long now = System.nanoTime();
            if (timestampNanos > now) {
         			....
                timestampNanos = now;
            }

            if (mHavePendingVsync) {
                ....log
            } else {
                mHavePendingVsync = true;
            }

            mTimestampNanos = timestampNanos;
            mFrame = frame;
            Message msg = Message.obtain(mHandler, this);
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
        }

        @Override
        public void run() {
            mHavePendingVsync = false;
            doFrame(mTimestampNanos, mFrame);
        }
    }
 ```
 主要实现:根据JNI层的 `timestampNanos`纳秒值计算成毫秒，通过 Handler 执行
 `Runable`，即执行了` doFrame(mTimestampNanos, mFrame);`这行代码。  
 
 ```java
 void doFrame(long frameTimeNanos, int frame) {
     final long startNanos;
     synchronized (mLock) {
     	if (!mFrameScheduled) {
         	return; // no work to do
     	}

     	long intendedFrameTimeNanos = frameTimeNanos;
     	startNanos = System.nanoTime();
     	final long jitterNanos = startNanos - frameTimeNanos;
     	if (jitterNanos >= mFrameIntervalNanos) {
        	 final long skippedFrames = jitterNanos / mFrameIntervalNanos;
         	if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
     				....
         	}
         	final long lastFrameOffset = jitterNanos % mFrameIntervalNanos;
         	....
         	frameTimeNanos = startNanos - lastFrameOffset;
      	}

      	if (frameTimeNanos < mLastFrameTimeNanos) {
				....
          	scheduleVsyncLocked();
          	return;
       	}

       	mFrameInfo.setVsync(intendedFrameTimeNanos, frameTimeNanos);
      	 	mFrameScheduled = false;
       	mLastFrameTimeNanos = frameTimeNanos;
    }

    try {
         Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Choreographer#doFrame");
         AnimationUtils.lockAnimationClock(frameTimeNanos / TimeUtils.NANOS_PER_MS);

         mFrameInfo.markInputHandlingStart();
         doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);

         mFrameInfo.markAnimationsStart();
         doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);

         mFrameInfo.markPerformTraversalsStart();
         doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);

         doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
     } finally {
         AnimationUtils.unlockAnimationClock();
         Trace.traceEnd(Trace.TRACE_TAG_VIEW);
     }
	.....
}
 ```
 与动画相关的关键代码: 
   
 ```java
 mFrameInfo.markAnimationsStart();
 doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);
 ```
 
 以下是 doCallbacks的实现，有没有感觉即将看见胜利的曙光？
 
 ```java
 void doCallbacks(int callbackType, long frameTimeNanos) {
     CallbackRecord callbacks;
     synchronized (mLock) {
         final long now = System.nanoTime();
         //取出可以执行的CallbackRecord 链表。
         callbacks = mCallbackQueues[callbackType].extractDueCallbacksLocked(
                    now / TimeUtils.NANOS_PER_MS);
         if (callbacks == null) {
             return;
         }
         mCallbacksRunning = true;
		  ....略去与动画无关的代码
        }
        try {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, CALLBACK_TRACE_TITLES[callbackType]);
            for (CallbackRecord c = callbacks; c != null; c = c.next) {
					.... log
					//循环执行链表
                c.run(frameTimeNanos);
            }
        } finally {
            synchronized (mLock) {
                mCallbacksRunning = false;
                do {
                    final CallbackRecord next = callbacks.next;
                    recycleCallbackLocked(callbacks);
                    callbacks = next;
                } while (callbacks != null);
            }
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }
 ```
 
 所以该方法最核心的功能是找到需要执行的 `CallbackRecord`链表，然后循环执行 它们的 `run`方法。
 
 ```java
 private static final class CallbackRecord {
        public CallbackRecord next;
        public long dueTime;
        public Object action; // Runnable or FrameCallback
        public Object token;

        public void run(long frameTimeNanos) {
            if (token == FRAME_CALLBACK_TOKEN) {
                ((FrameCallback)action).doFrame(frameTimeNanos);
            } else {
                ((Runnable)action).run();
            }
        }
    }
 ```
 在前面 注册回调`postFrameCallbackDelayed`时 token 是 `FRAME_CALLBACK_TOKEN` 所以执行 run 方法中的第一个if 分支`    ((FrameCallback)action).doFrame(frameTimeNanos);`
  回调 	`FrameCallback.doFrame(frameTimeNanos)` 方法。
  下面是 `Choreographer.FrameCallback` 的实现
  
  ```java
  private final Choreographer.FrameCallback mFrameCallback = new Choreographer.FrameCallback() {
        @Override
        public void doFrame(long frameTimeNanos) {
            doAnimationFrame(getProvider().getFrameTime());
            if (mAnimationCallbacks.size() > 0) {
                getProvider().postFrameCallback(this);
            }
        }
    };
  ```
 看到`doAnimationFrame(getProvider().getFrameTime());`这一行代码，我们已经确认，我们的动画逻辑即将开始了。
 我们先搁置`doAnimationFrame`这行代码，先分析完 `doFrame` 回调。  
 
 ```java
 if (mAnimationCallbacks.size() > 0) {
     getProvider().postFrameCallback(this);
 }
 ```
 这里如果 `mAnimationCallbacks.size >0`会再次将该 callback 注册给 `Choreographer`。为什么还要注册一次呢？之前不是注册过一次了吗？难道`Choreographer`把这个callback 释放了。  
 我们回到`Choreographer.doCallbacks`方法。
 有一段 final 括号体中的代码我们没有分析  
 
 ```java
 finally {
    synchronized (mLock) {
        mCallbacksRunning = false;
        do {
            final CallbackRecord next = callbacks.next;
            recycleCallbackLocked(callbacks);
            callbacks = next;
        } while (callbacks != null);
    }
    Trace.traceEnd(Trace.TRACE_TAG_VIEW);
}
 ```
 这里调用了`recycleCallbackLocked(callbacks);`来回收整个链表中的节点记录。
 
 ```java
 private void recycleCallbackLocked(CallbackRecord callback) {
    callback.action = null;
    callback.token = null;
    callback.next = mCallbackPool;
    mCallbackPool = callback;
 }
 ```
 当 `mAnimationCallbacks.size >0`时 需要再次添加回调，以便来获取下一帧的回调。
 当动画 paused 或 end 后 mAnimationCallbacks 会相应的remove callback。
 
### 小结1
到目前为止 我们知道了当调用 `ObjectAnimator.start()`后:  
1. 取消之前的动画对相同属性,相同target的动画，防止出现多个动画同时更新 Target 的属性，出现错乱。不过这一行为默认是关闭的,设置`ObjectAnimator.setAutoCancel(true)`来打开;  
2. 执行 `ValueAnimator.start()`方法；  
3. `AnimationHandler.addAnimationFrameCallback`向`Choreographer`注册`Choreographer.FrameCallback`回调,通过该回调获得渲染时间脉冲的回调;  
4. 通过系统的`vsync`垂直同步信号来协调 cpu,gpu 和渲染的时序;  
5. `Choreographer` 获得 `vsync`信号后 根据 当前帧的`纳秒`来查找哪些 `Choreographer.FrameCallback`会被执行。  
6. 执行`AnimationHandler.doAnimationFrame()`方法，开始真正的动画逻辑。

### 动画属性更新
刚才我们分析到 `AnimationHandler.doAnimationFrame()` 方法，现在看看这个方法的功能是什么?  

```java
private void doAnimationFrame(long frameTime) {
   int size = mAnimationCallbacks.size();
   long currentTime = SystemClock.uptimeMillis();
   for (int i = 0; i < size; i++) {
        final AnimationFrameCallback callback = mAnimationCallbacks.get(i);
        if (callback == null) {
            continue;
        }
        if (isCallbackDue(callback, currentTime)) {
            callback.doAnimationFrame(frameTime);
            if (mCommitCallbacks.contains(callback)) {
                getProvider().postCommitCallback(new Runnable() {
                   @Override
                   public void run() {
                      commitAnimationFrame(callback, getProvider().getFrameTime());
                   }
                });
             }
        }
   }//for
   cleanUpList();
}
```
该方法主要作用是遍历所有的 `AnimationFrameCallback`。 这个`AnimationFrameCallback` 是什么？什么时候添加到`mAnimationCallbacks`的？  
刚才分析`Choreographer.FrameCallback.doFrame`时提到过，在动画paused 或 end 时会将`AnimationFrameCallback` 从`mAnimationCallbacks `中移除，如果`mAnimationCallbacks`的size 为0 就不再向`Choreographer`注册 callback。也就代表没有动画要被执行了。  
其实 `AnimationFrameCallback`就是 `ObjectAnimator`或`ValueAnimaor`本身。我们看看 `ValueAnimator`的定义:

```java
public class ValueAnimator extends Animator implements AnimationHandler.AnimationFrameCallback {
}
```
所以 这些`AnimationFrameCallback`就是那些待执行的 `属性动画`。  
接下来看看`isCallbackDue(callback, currentTime)`这个 `if` 判断:

```java
private boolean isCallbackDue(AnimationFrameCallback callback, long currentTime) {
   Long startTime = mDelayedCallbackStartTime.get(callback);
   if (startTime == null) {
       return true;
    }
    if (startTime < currentTime) {
        mDelayedCallbackStartTime.remove(callback);
        return true;
    }
    return false;
}
```
根据 当前的的 纳秒时间 判断 动画是否需要执行，因为有些动画做了`delay`可能在当前的frame 窗口不需要执行。  
`callback.doAnimationFrame(frameTime);` 这行代码 开始回调 `ValueAnimator.doAnimationFrame`方法。然后判断`if (mCommitCallbacks.contains(callback))` 为 true的话会再次向`Choreographer`注册一个`Runnable`的callback，当下一个 frame 时间到来时 执行 `Runnable`。  

我们关注**重点代码**`callback.doAnimationFrame(frameTime)` 也即是`ValueAnimator.doAnimationFrame(frameTime)`

```java
public final void doAnimationFrame(long frameTime) {
   AnimationHandler handler = AnimationHandler.getInstance();
   if (mLastFrameTime == 0) {
       // First frame
       //如果是动画的第一次回调，注册调整到下一个 frame 窗口 再执行。
       handler.addOneShotCommitCallback(this);
       if (mStartDelay > 0) {
           startAnimation();
       }
       if (mSeekFraction < 0) {
           mStartTime = frameTime;
       } else {
           long seekTime = (long) (getScaledDuration() * mSeekFraction);
           mStartTime = frameTime - seekTime;
           mSeekFraction = -1;
       }
       mStartTimeCommitted = false; // allow start time to be compensated for jank
    }
    mLastFrameTime = frameTime;
    if (mPaused) {
        mPauseTime = frameTime;
        handler.removeCallback(this);
        return;
    } else if (mResumed) {
        mResumed = false;
        if (mPauseTime > 0) {
            // Offset by the duration that the animation was paused
            mStartTime += (frameTime - mPauseTime);
            mStartTimeCommitted = false; // allow start time to be compensated for jank
        }
        handler.addOneShotCommitCallback(this);
   }
   ... comments
   final long currentTime = Math.max(frameTime, mStartTime);
   boolean finished = animateBasedOnTime(currentTime);
   if (finished) {
       endAnimation();
   }
}
```
当时第一帧动画时 会调整到下一个 frame 窗口执行。如果这个动画是 delay 动画，会执行`startAnimation()`初始化动画，标记动画正在 `mRunning`，并且 对`PropertyValuesHolder`执行初始化操作--主要就是初始化`估值器`。  
下面分析 ` boolean finished = animateBasedOnTime(currentTime);`
返回值 `finished`标记是否动画已经执行完毕。如果最后一个关键帧(Keyframe)执行完毕,这里返回true，会执行`endAnimation()`做一些状态位复位和动画结束回调等等。

```java
boolean animateBasedOnTime(long currentTime) {
   boolean done = false;
   if (mRunning) {
       final long scaledDuration = getScaledDuration();
       final float fraction = scaledDuration > 0 ?
                    (float)(currentTime - mStartTime) / scaledDuration : 1f;
       final float lastFraction = mOverallFraction;
       final boolean newIteration = (int) fraction > (int) lastFraction;
       final boolean lastIterationFinished = (fraction >= mRepeatCount + 1) &&
                    (mRepeatCount != INFINITE);
       if (scaledDuration == 0) {
           // 0 duration animator, ignore the repeat count and skip to the end
           done = true;
       } else if (newIteration && !lastIterationFinished) {
          // Time to repeat
          if (mListeners != null) {
              int numListeners = mListeners.size();
              for (int i = 0; i < numListeners; ++i) {
                   mListeners.get(i).onAnimationRepeat(this);
              }
           }
       } else if (lastIterationFinished) {
           done = true;
       }
       mOverallFraction = clampFraction(fraction);
       float currentIterationFraction = getCurrentIterationFraction(mOverallFraction);
       animateValue(currentIterationFraction);
   }
   return done;
}
```

`currentTime`是`Choreographer`发出的计时脉冲时间，纳秒计时。
根据 `currentTime` 计算 `fraction`系数，即动画时间流逝比。  
然后执行`animateValue(currentIterationFraction)` 计算动画的在当前时间比例下属性动画的值，如果是 `ObjectAnimator`还会降属性值设置该`Target`。  
`animateValue`方法被 `ObjectAnimator` 重载了。我们先分析`ValueAnimator. animateValue(fraction)` 然后再分析`ObjectAnimator. animateValue(fraction)`

```java
void animateValue(float fraction) {
   fraction = mInterpolator.getInterpolation(fraction);
   mCurrentFraction = fraction;
   int numValues = mValues.length;
   for (int i = 0; i < numValues; ++i) {
        mValues[i].calculateValue(fraction);
   }
   if (mUpdateListeners != null) {
       int numListeners = mUpdateListeners.size();
       for (int i = 0; i < numListeners; ++i) {
            mUpdateListeners.get(i).onAnimationUpdate(this);
       }
   }
}
```
参数 `fraction`这个时间流逝比系数 是线性的。通过`mInterpolator.getInterpolation()`计算出我们想要的fraction。然后使用这个系数计算 `PropertyValuesHolder.calculateValue(fraction)`。  
计算完 属性值后 执行 `mUpdateListeners`的更新操作。到目前为止我们终于知道我们经常使用的 `AnimatorUpdateListener.onAnimationUpdate（）`何时执行的了。

我们再分析 被 `ObjectAnimator`重载后的`animateValue(fraction)`方法

```java
void animateValue(float fraction) {
   final Object target = getTarget();
   if (mTarget != null && target == null) {
    // We lost the target reference, cancel and clean up. Note: we allow null target if the
    /// target has never been set.
    cancel();
    return;
    }

    super.animateValue(fraction);
    int numValues = mValues.length;
     for (int i = 0; i < numValues; ++i) {
          mValues[i].setAnimatedValue(target);
     }
}
```
主要功能是 通过调用 `super.animateValue()`计算属性的值。然后调用`PropertyValuesHolder.setAnimatedValue(Object)`来更新属性值到对应的`Target`上。

[深入理解Android属性动画的实现-1](http://aicoding.tech/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Android%E5%B1%9E%E6%80%A7%E5%8A%A8%E7%94%BB%E7%9A%84%E5%AE%9E%E7%8E%B0-1/) 这篇文章中介绍过 `PropertyValuesHolder`具有更新属性的能力，也持有`关键帧`的引用。

```java
void setAnimatedValue(Object target) {
   if (mProperty != null) {
       mProperty.set(target, getAnimatedValue());
   }
   if (mSetter != null) {
       try {
         mTmpValueArray[0] = getAnimatedValue();
         mSetter.invoke(target, mTmpValueArray);
       } catch (InvocationTargetException e) {
                Log.e("PropertyValuesHolder", e.toString());
       } catch (IllegalAccessException e) {
                Log.e("PropertyValuesHolder", e.toString());
       }
  }
}
```

通过 `Property`更新属性值。如果之前是通过 `propertyName`来初始化的动画，这里通过 `mSetter`来反射调用 set 方法，更新属性值。

[深入理解Android属性动画的实现-1](http://aicoding.tech/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Android%E5%B1%9E%E6%80%A7%E5%8A%A8%E7%94%BB%E7%9A%84%E5%AE%9E%E7%8E%B0-1/#Keyframe) 这篇文章还介绍了如何通过`fraction`计算动画的属性值。这里就不在赘述。 

### 总结
到目前位置我们已经分析完 属性动画的创建---> 属性动画的启动 --> 属性的计算和更新。  
这里我们再回顾一下:
1. `ObjectAnimator.start()`开始启动动画；
2. 向`Choreographer`注册callback；
3.  `Choreographer`获得`vsync`垂直同步信号量后回调`Choreographer.FrameCallback.doFrame()`执行逻辑进入到`AnimationHandler`中；
4. `AnimationHandler`持有 `AnimationFrameCallback`也即是`ValueAnimator`,然后执行`ValueAnimator.doAnimationFrame(time)`;
5. `ValueAnimator.animateBasedOnTime(time)`执行，通过 `TimeInterpolator`计算最终的 时间流逝比`fraction`,然后调用`PropertyValuesHolder.calculateValue(fraction)`计算属性的值，并回调 `AnimatorUpdateListener.onAnimationUpdate()`方法。
6. `PropertyValuesHolder`调用`Keyframes.getIntValue(fraction)`,这中间又使用到估值器`TypeEvaluator`和`Keyframe`最终结算处我们需要的属性值。
7. 然后`ObjectAnimator`调用`PropertyValuesHolder.setAnimatedValue(target)`来更新 `target`的属性值。

### 附加知识点
文章中提到过`Choreographer`和`Android黄油计划`。其实在本文的流程分析中已经简单分析了 `Choreographer`在动画类型上的执行流程：  
1. 创建`DisplayEventReceiver`的子类`FrameDisplayEventReceiver`来与JNI 层交互。JNI层的 `vsync`信号量通过callback 这个类的 `dispatchVsync`方法来告诉应用层 可以开始新的一帧的渲染了。  
2. `Choreographer`接收到 `vsync`信号后执行`doFrame(frameTimeNanos,frame)`方法，对三个支持的类型`CALLBACK_INPUT`,`CALLBACK_ANIMATION`,`CALLBACK_TRAVERSAL`做相应的回调。  
**需要说明下:**  _源码中还有一个类型`CALLBACK_COMMIT`主要处理注册 commit Runnable的需求，即延迟一个frame 的需求。因为它是一种业务辅助，不像其它三种，有明显的业务支持。所以在本文中倾向说三种支持类型，请知悉_。  
3. 回调进入`Choreographer.FrameCallback.doFrame(timeNanos)`  
4. 然后进入到业务层，例如 :  
`CALLBACK_ANIMATION`类型进入到`AnimationHandler.mFrameCallback`;  
`CALLBACK_TRAVERSAL`类型进入到`ViewRootImpl` 执行`scheduleTraversals()` 进而执行了`View.layout`,`View.measure`,`View.draw`方法开启View的渲染操作；  `CALLBACK_INPUT`这个类型相比前两个特殊一些，因为输入事件由另一个引擎负责。让输入引擎产生输入事件后不是立刻在 视图层产生响应。而是要等待下一个 `vsync`垂直同步信号，跟着统一的时钟脉冲来响应。所以 在`ViewRootImpl`中会使用到`mChoreographer.postCallback(Choreographer.CALLBACK_INPUT,mConsumedBatchedInputRunnable, null)`。具体这里不再展开描述。

简言之，`Choreographer`就是一个消息处理器，根据 `vsync`垂直同步信号 来处理三种支持类型的回调。

至于 `Android黄油计划(Project Butter)` `Choreographer`只是其中一个重要的特性。还有其他方便的优化。例如 引入了`三重缓冲`和 JNI层的`vsync`。至于 `vsync`的好处以及和 该计划之前 的Android的渲染相比 请参考[Android 4.4 Graphic系统详解（2） VSYNC的生成](http://blog.csdn.net/michaelcao1980/article/details/43233765) 这篇优质文章。在这里也对该篇文章的愿作者致敬。



 


