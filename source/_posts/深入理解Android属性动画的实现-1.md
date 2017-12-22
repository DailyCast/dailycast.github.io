---
title: 深入理解Android属性动画的实现-1
date: 2017-11-25 22:07:15
tags: 
	- Android
	- 动画
category: Android
comments: true
---
### 导读
在文章开头我们先简单回忆一下常用的属性动画 `API` 有哪些？  
1.动画相关的类:
`ObjectAnimator`,`ValueAnimator`,`Property`,`PropertyValuesHolder`,`Keyframe`,`Keyframes`,`TimeInterpolator`,`TypeEvaluator`
在剖析过程中我们会解析相关类的作用和实现方式  
2.常用的 `属性动画` 方法:

```java
ObjectAnimator.ofInt()
ObjectAnimator.ofFloat()
ObjectAnimator.ofArgb()
ObjectAnimator.ofObject()
ObjectAnimator.ofPropertyValuesHolder()
```
下面我们以一个常用的方法 `ObjectAnimator.ofInt()` 为切入点层层深入解析属性动画的创建过程。
### 动画的创建
早期版本的属性动画 `ofInt` 方法有两个实现，到Android O 为止 已经有 4 个实现了。
主要就是增加了 `Path` 参数。我们这里并不去解析新方法，因为它的实现方式和最基本的方法相同。

```java
public static ObjectAnimator ofInt(Object target, String propertyName, int... values) {
        ObjectAnimator anim = new ObjectAnimator(target, propertyName);
        anim.setIntValues(values);
        return anim;
    }
public static <T> ObjectAnimator ofInt(T target, Property<T, Integer> property, int... values) {
        ObjectAnimator anim = new ObjectAnimator(target, property);
        anim.setIntValues(values);
        return anim;
    }
```
_注意:我这里以Android-25 源码为代码源。_  
上述代码中`ofInt` 方法的第一行均是 `创建 ObjectAnimator 对象`。但分别使用了两种构造方式。
下面我们探索一下 `ObjectAnimator ` 的构造方法:

```java
//无参构造
public ObjectAnimator() {
}
//指定 propertyName
private ObjectAnimator(Object target, String propertyName) {
  setTarget(target);
  setPropertyName(propertyName);
}
//指定 Property
 private <T> ObjectAnimator(T target, Property<T, ?> property) {
    setTarget(target);
    setProperty(property);
    }
```
无参构造方法没有什么可说的。另外两个构造方法第一行代码都是

```java
setTarget(target);
```
该`setTarget`方法的作用是记录需要做动画的对象，方便以后对该对象的相关属性做更改。

1.我们首先看看 `propertyName ` 参数的构造方法的`setPropertyName ` 方法

```java
 public void setPropertyName(@NonNull String propertyName) {
        if (mValues != null) {
            PropertyValuesHolder valuesHolder = mValues[0];
            String oldName = valuesHolder.getPropertyName();
            valuesHolder.setPropertyName(propertyName);
            mValuesMap.remove(oldName);
            mValuesMap.put(propertyName, valuesHolder);
        }
        mPropertyName = propertyName;
        mInitialized = false;
    }
```
我们这里的调用情形，`mValues `肯定为null，不会走 `if`分支，剩下的就是简单的记录我们设置的`propertyName`用`mPropertyName `保持。如果`mValues `不为空的情况下，会更新 `mValues`的第一个元素的属性值，并更新对应的map集合  
2.我们再看看`Property `参数的构造方法的`setProperty`方法，同样也是简单的记录`mProperty`和`mPropertyName`  
*那么 这个`Property `是何方神圣？*
### Property 解析
```java
public abstract class Property<T, V> {
    private final String mName;
    private final Class<V> mType;
    public static <T, V> Property<T, V> of(Class<T> hostType, Class<V> valueType, String name) {
        return new ReflectiveProperty<T, V>(hostType, valueType, name);
    }
    public Property(Class<V> type, String name) {
        mName = name;
        mType = type;
    }
    public boolean isReadOnly() {
        return false;
    }
    public void set(T object, V value) {
        throw new UnsupportedOperationException("Property " + getName() +" is read-only");
    }
    public abstract V get(T object);
    public String getName() {
        return mName;
    }
    public Class<V> getType() {
        return mType;
    }
}
```
这是一个 `abstract`修饰的类。这个类很简单，就是定义了一些需要被子类实现的方法。*(为了减小博客的字数，我这里隐去源码中的相关注释，如有需要请自行到源码中查看)*
通过对注释的理解我们知道 这个类的注意功能就是`代表`需要动画的`属性`,实现这个类有一些先决条件需要满足: 
必须满足一下三个条件中的一个   
1.被 `Property ` 代表的类的属性必须有一个`public`的无参的`get`方法，和一个可选的`set`方法，`set`方法的参数和`get`方法的返回值一个类型  
2.有一个`public`的无参的`is` 方法，和一个可选的set方法 set方法的参数和is方法的返回值一个类型  
3.一个`public`的 属性 `name`。  
假如有 `get/is`方法，但是没有`set`方法，那么这个 `Property` 是一个只读的属性。
`set` 方法必须实现，都在抛出异常
到目前为止我们了解了 动画初始化时会初始化 `mPropertyName`变量，记录属性的名称；或初始化`mProperty`，来保持对属性的访问能力和更新能力。  
*这里说明一下，常规情况下需要满足这三个中一个即可,如果我们又其他方式来变量属性值，而且也在 `Property` 中做了相应的映射关系，不满足也可以。这个在之后文章中自定义属性动画中可以看出*  
我们回到之前的 `ofInt`方法
紧接着 执行了 代码

```java
 anim.setIntValues(values);
```
这行代码看似简单，其实包含了很多信息。从字面意思理解，就是保存动画的属性值范围。

```java
@Override
    public void setIntValues(int... values) {
        if (mValues == null || mValues.length == 0) {
            if (mProperty != null) {
                setValues(PropertyValuesHolder.ofInt(mProperty, values));
            } else {
                setValues(PropertyValuesHolder.ofInt(mPropertyName, values));
            }
        } else {
            super.setIntValues(values);
        }
    }
```
第一次调用 `mValues `为 `null`走入`if`流程，假如我们调用的 `propertyName`参数的方法，会执行`setValues(PropertyValuesHolder.ofInt(mPropertyName, values));`,反之执行`setValues(PropertyValuesHolder.ofInt(mProperty, values));`  
我们先看看`PropertyValuesHolder.ofInt(mProperty, values)`

```java
 public static PropertyValuesHolder ofInt(Property<?, Integer> property, int... values) {
        return new IntPropertyValuesHolder(property, values);
}
 
public IntPropertyValuesHolder(Property property, int... values) {
            super(property);
            setIntValues(values);
            if (property instanceof  IntProperty) {
                mIntProperty = (IntProperty) mProperty;
            }
 }
 @Override
public void setIntValues(int... values) {
            super.setIntValues(values);
            mIntKeyframes = (Keyframes.IntKeyframes) mKeyframes;
}
 public void setIntValues(int... values) {
        mValueType = int.class;
        mKeyframes = KeyframeSet.ofInt(values);
 }        
```
该方法主要的作用就是创建一个`PropertyValuesHolder`,保持属性`Property`
并将 属性值保存成`关键帧`，我们关注核心如何生成关键帧 
 
```java
public static KeyframeSet ofInt(int... values) {
        int numKeyframes = values.length;
        IntKeyframe keyframes[] = new IntKeyframe[Math.max(numKeyframes,2)];
        if (numKeyframes == 1) {
            keyframes[0] = (IntKeyframe) Keyframe.ofInt(0f);
            keyframes[1] = (IntKeyframe) Keyframe.ofInt(1f, values[0]);
        } else {
            keyframes[0] = (IntKeyframe) Keyframe.ofInt(0f, values[0]);
            for (int i = 1; i < numKeyframes; ++i) {
                keyframes[i] =
                        (IntKeyframe) Keyframe.ofInt((float) i / (numKeyframes - 1), values[i]);
            }
        }
        return new IntKeyframeSet(keyframes);
    }
```
上述代码是生成关键帧的逻辑:
1.判断值的个数，如果只有一个值。那么分配长度为2的`IntKeyframe `数组,并将值保持在第二个位置。
2.如果多于一个值，会分配 值个数 长度的数组，将每一个数值都保持在数组对应的位置。
需要注意的是:除了第一帧，其他帧都分配了`fraction`,这个系数有这些数字的传入顺序决定。假如我们传入3个值，第一帧的系数默认为0，第二帧的系数为0.5，最后一个值得系数是1
。这个系数在后来动画的执行中为生成相应的属性值起到关键作用。

到目前为止，我们知道`PropertyValuesHolder `持有了`Property `并将动画的值保存成关键帧，并持有这些关键帧。

同样`setValues(PropertyValuesHolder.ofInt(mPropertyName, values));`方法也相同的方式生成了`PropertyValuesHolder `并持有 生成的关键帧

然后我们分析一下`setValues`方法。
这是父类 `ValueAnimator`的方法

```java
public void setValues(PropertyValuesHolder... values) {
        int numValues = values.length;
        mValues = values;
        mValuesMap = new HashMap<String, PropertyValuesHolder>(numValues);
        for (int i = 0; i < numValues; ++i) {
            PropertyValuesHolder valuesHolder = values[i];
            mValuesMap.put(valuesHolder.getPropertyName(), valuesHolder);
        }
        // New property/values/target should cause re-initialization prior to starting
        mInitialized = false;
    }
```
主要功能：  
1.保存前面生成的 `PropertyValuesHolder `,用 `mValues`持有引用。  
2.将 `PropertyValuesHolder `数组按照属性名为key，`PropertyValuesHolder `为值保存至map中，并标记为初始化` mInitialized = false`,在下一篇文章中会提到何时将标记位`mInitialized `置为 `true`

到目前为止我们已经分析完了 `ObjectAnimator.ofInt`方法。  
其中有几个未解释到的类:  
1.PropertyValuesHolder  
2.Keyframe

### PropertyValuesHolder
由前面的分析我们大致知道 `PropertyValuesHolder ` 持有了 `Property`或`mPropertyName`,以及持有 关键帧。  
除此之外，它还有 属性`mSetter`,`mGetter`,`mEvaluator`
```java
Method mSetter = null;
private Method mGetter = null;
private TypeEvaluator mEvaluator;
```
以及一堆 `of`方法 
	
	ofInt
	ofMultiInt
	ofFloat
	ofMultiFloat
	ofObject
	ofKeyframe
	
这些 `of`方法我们前面解析过 `ofInt` 这里不再展开解释。
看到 `mSetter`和`mGetter`方法，我们联想到这个类肯定会操作 动画对象的属性。果不其然，让我们找到了 `setupSetter`和`setupGetter` 方法

```java
 void setupSetter(Class targetClass) {
        Class<?> propertyType = mConverter == null ? mValueType : mConverter.getTargetType();
        mSetter = setupSetterOrGetter(targetClass, sSetterPropertyMap, "set", propertyType);
    }

    /**
     * Utility function to get the getter from targetClass
     */
private void setupGetter(Class targetClass) {
        mGetter = setupSetterOrGetter(targetClass, sGetterPropertyMap, "get", null);
    }
```
这两个方法通过 拼接`set` 和`get`，通过反射获取属性的 get set 方法，从而获得对象属性的控制权。
当然 一些`PropertyValuesHolder`的子类重载使用了 jni 方法获得反射方法。其实都是目的都是一样。

当然，获得了这些属性的控制权，自然会更新这些属性

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
这个 `setAnimatedValue `方法就是用于更新这些属性；如果有 `mProperty`就通过 `Property`更新属性，如果没有的话，就通过反射更新。
当然在更新属性之前需要计算出正确的属性值。

```java
 void calculateValue(float fraction) {
        Object value = mKeyframes.getValue(fraction);
        mAnimatedValue = mConverter == null ? value : mConverter.convert(value);
    }
```
`calculateValue `方法就是用于计算这些值。计算又用到了`Keyframes`，下面分析关键帧相关的API。到此为止我们理解了 `PropertyValuesHolder `的作用

总结一下：  
1.持有 `Property` 用于获取,更新属性的能力。  
2.持有 `mPropertyName` 通过反射获取`get` , `set` 方法，获取更新属性的能力  
3.计算属性的值  
4.更新属性    
所以 `PropertyValuesHolder`就是控制动画对象属性的关键。所以也很容易理解，为什么所有的属性动画的value都转化成`PropertyValuesHolder `来操作.

### Keyframe
下面我们在详细了解下何为关键帧，它是如何运作的?    
先看几个关于`关键帧`的类 `Keyframe`,`Keyframes` 以及它们的子类  
`Keyframe`顾名思义就是记录关键帧数据,他默认有`三个`实现类`IntKeyframe`,`FloatKeyframe`,`ObjectKeyframe`,当然我们也能继承`Keyframe`实现自己的关键帧类型。不过大部分情况，提供的这三种方式已经够用了。  
`Keyframe` 有三个重要的`属性值`:  
1.`mHasValue` 用于记录关键帧是否初始化了值，以 子类 **IntKeyframe** 为例，它有两个构造方法:

```java
IntKeyframe(float fraction, int value) {
            mFraction = fraction;
            mValue = value;
            mValueType = int.class;
            mHasValue = true;
        }

IntKeyframe(float fraction) {
            mFraction = fraction;
            mValueType = int.class;
        }
```
第一个构造初始化了 `mValue` ，同时 `mHasValue `会标记 为`true`;
第二个构造方法 只初始化了 `mFraction `，并未给 `mValue`赋值，默认 `mHasValue `为`false`，假如我们使用了`mHasValue `为 `false`的 关键帧。那么在动画初始化时会调用`PropertyValuesHolder.setupSetterAndGetter`给每一个关键帧赋初始值。

2.`mFraction`记录该关键帧所有关键帧中的位置，`float`类型，值范围`0 - 1`   
3.`mValue`记录value值，当然不同类型的`Keyframe` value值类型也不同，所以 `mValue`由子类实现

### Keyframes
我们可以认为 `Keyframes`是一堆 `Keyframe`组成的集合。  
先看看继承体系:  
它的实现类有`KeyframeSet`,`PathKeyframes`,它的子接口 有`IntKeyframes`,`FloatKeyframes`  
我们看看常用的`KeyframeSet`,继承它的两个子类`IntKeyframeSet`,`FloatKeyframeSet`,通知实现了对应的`IntKeyframes`和`FloatKeyframes `接口。

其实 `Keyframes`最核心的方法是`getValue`方法，其子实现均是为了实现它。下面是`KeyframeSet`的`getValue`方法的实现：

```java
public Object getValue(float fraction) {
        if (mNumKeyframes == 2) {
            if (mInterpolator != null) {
                fraction = mInterpolator.getInterpolation(fraction);
            }
            return mEvaluator.evaluate(fraction, mFirstKeyframe.getValue(),
                    mLastKeyframe.getValue());
        }
        if (fraction <= 0f) {
            final Keyframe nextKeyframe = mKeyframes.get(1);
            final TimeInterpolator interpolator = nextKeyframe.getInterpolator();
            if (interpolator != null) {
                fraction = interpolator.getInterpolation(fraction);
            }
            final float prevFraction = mFirstKeyframe.getFraction();
            float intervalFraction = (fraction - prevFraction) /
                (nextKeyframe.getFraction() - prevFraction);
            return mEvaluator.evaluate(intervalFraction, mFirstKeyframe.getValue(),
                    nextKeyframe.getValue());
        } else if (fraction >= 1f) {
            final Keyframe prevKeyframe = mKeyframes.get(mNumKeyframes - 2);
            final TimeInterpolator interpolator = mLastKeyframe.getInterpolator();
            if (interpolator != null) {
                fraction = interpolator.getInterpolation(fraction);
            }
            final float prevFraction = prevKeyframe.getFraction();
            float intervalFraction = (fraction - prevFraction) /
                (mLastKeyframe.getFraction() - prevFraction);
            return mEvaluator.evaluate(intervalFraction, prevKeyframe.getValue(),
                    mLastKeyframe.getValue());
        }
        Keyframe prevKeyframe = mFirstKeyframe;
        for (int i = 1; i < mNumKeyframes; ++i) {
            Keyframe nextKeyframe = mKeyframes.get(i);
            if (fraction < nextKeyframe.getFraction()) {
                final TimeInterpolator interpolator = nextKeyframe.getInterpolator();
                final float prevFraction = prevKeyframe.getFraction();
                float intervalFraction = (fraction - prevFraction) /
                    (nextKeyframe.getFraction() - prevFraction);
                // Apply interpolator on the proportional duration.
                if (interpolator != null) {
                    intervalFraction = interpolator.getInterpolation(intervalFraction);
                }
                return mEvaluator.evaluate(intervalFraction, prevKeyframe.getValue(),
                        nextKeyframe.getValue());
            }
            prevKeyframe = nextKeyframe;
        }
        // shouldn't reach here
        return mLastKeyframe.getValue();
    }
```
该方法核心是 通过参数`fraction`(时间流逝比)来计算对应的属性值，计算出来的值如无`TypeConverter`转换需求，那就会把这个值设置到 动画`Target`的属性上，从而实现属性动画对属性的更改。  
观察到这里还使用了一个`TypeEvaluator`,这哥们大家估计通过其他途径都了解过，我们经常把 `TimeInterpolator `和`TypeEvaluator `一起来解释属性动画的原理，即插值器和估值器对属性动画的作用。我们这里不展开讨论，只基于源码去分析。  
我们先看第一个`if`分支  

```java
if (mNumKeyframes == 2) {
   if (mInterpolator != null) {
      fraction = mInterpolator.getInterpolation(fraction);
   }
   return mEvaluator.evaluate(fraction, mFirstKeyframe.getValue(),
                    mLastKeyframe.getValue());
}
```
对应当仅有两个关键帧时的情况。举个例子，假如我们调用`ObjectAnimator.ofInt`方法只传入了一个`int`值，这里的 `mNumKeyframes`会是2,而且看注释，只有在这种情况下才使用`mInterpolator`-插值器。通过插值器 得到`修正后的系数fraction `，然后拿这个系数，通过`估值器mEvaluator`计算出属性值。我们发现估值器是何等的重要，必须要被赋值的，否则无法计算出属性值。  
继续 ` if (fraction <= 0f) `和` if (fraction >= 1f)`这两个分支是特殊情况，出现在动画第一帧和最后一帧的情况。  
下面看看正常动画中的分支流程：
 
```java
Keyframe prevKeyframe = mFirstKeyframe;
        for (int i = 1; i < mNumKeyframes; ++i) {
            Keyframe nextKeyframe = mKeyframes.get(i);
            if (fraction < nextKeyframe.getFraction()) {
                final TimeInterpolator interpolator = nextKeyframe.getInterpolator();
                final float prevFraction = prevKeyframe.getFraction();
                float intervalFraction = (fraction - prevFraction) /
                    (nextKeyframe.getFraction() - prevFraction);
                // Apply interpolator on the proportional duration.
                if (interpolator != null) {
                    intervalFraction = interpolator.getInterpolation(intervalFraction);
                }
                return mEvaluator.evaluate(intervalFraction, prevKeyframe.getValue(),
                        nextKeyframe.getValue());
            }
            prevKeyframe = nextKeyframe;
        }
```
通过`fraction`查找符合`fraction < nextKeyframe.getFraction()`的第一帧，然后计算 `fraction`在前一帧到当前帧范围内的位置，例如查找到符合条件的当前帧 fraction =0.5;前一帧fraction = 0.2, 动画当前的 fraction =0.3，那么新的 intervalFraction = 1/3f。然后拿着这个值给估值器计算属性值。

到目前为止我们知道了，关键帧的作用如下:  
1.保存每个关键帧的值  
2.保存每个关键帧的 fraction  
3.通过动画类传入的 fraction，利用估值器计算出 真正的属性值。

那么剩下的就是如何将这些计算出来的`属性值`设置到对应的`Target`上了。这个问题我们下一篇分析动画的启动原理时会再分析。

### 回顾
1.以`ObjectAnimator.ofInt`为切入点 分析 ObjectAnimator对象的创建  
2.属性名称`propertyName`和`Property`的作用:  
	propertyName 用于拼接 get/set方法，获得控制属性的能力;
	Property 利用自己的`get`和`set`方法直接控制属性。
3. `PropertyValuesHolder`的作用:通过 propertyName 或 Property 获取更新属性的能力;根据动画时间流逝比计算属性的值;更新属性值到对应的`Target`上  
4. 关键帧 `keyframe`用于保存动画的每一个关键数据帧，个数由传入的 value 个数决定,系数 fraction 由 value 传入顺序决定。  
5. `keyframes`以及它的子类`KeyframeSet`的作用:利用估值器和 fraction 计算出应该更新到 `Target`上的属性值。

基本API 解析完毕，下一篇我们会分析动画如何开始的，fraction系数如何得到的，以及何时更新 属性值。




