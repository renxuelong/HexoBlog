---
layout: post
title: LayoutInflater源码解析
date: 2018-11-27 17:18:14
categories: 
- Andriod源码

tags:
- Android
- 源码解析
- 高级UI
---

LayoutInflater 在开发中是经常使用的一个类，一般我们都通过 from 方法获取 LayoutInflater 的实例，并通过其 inflate 方法解析 xml 文件。这么常用与重要的类值得我们去追着源码更深程度的去理解 xml 文件的加载和解析过程。

## 一、LayoutInflater 工作过程

```Java
public static LayoutInflater from(Context context) {
    LayoutInflater LayoutInflater =
            (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
    if (LayoutInflater == null) {
        throw new AssertionError("LayoutInflater not found.");
    }
    return LayoutInflater;
}
```

关于 LayoutInlnflater 的 from 方法，我们可以通过 [这篇博客](http://www.jianshu.com/p/14d8fe11d22f) 看一下，讲了通过 Context 构建 LayoutInflater 的过程。

这篇博客中提到了 LayoutInflater 的 from 方法最后返回的是一个 PhoneLayoutInflater 类的对象，该类继承了 LayoutInflater 并实现了其抽象方法

接着我们来分析 LayoutInflater 的 inflate 方法的工作过程


<!-- more -->


## 二、LayoutInflater 的 inflate 方法的工作过程

```Java
// LayoutInflater
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
    final Resources res = getContext().getResources();
    if (DEBUG) {
        Log.d(TAG, "INFLATING from resource: \"" + res.getResourceName(resource) + "\" ("
                + Integer.toHexString(resource) + ")");
    }

    // 由 Resources 构建布局的解析器
    final XmlResourceParser parser = res.getLayout(resource);
    try {
        // 解析布局文件，并将结果返回
        return inflate(parser, root, attachToRoot);
    } finally {
        parser.close();
    }
}

// LayoutInflater
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    synchronized (mConstructorArgs) {
    
        final Context inflaterContext = mContext; // 存储 Context
        final AttributeSet attrs = Xml.asAttributeSet(parser);
        Context lastContext = (Context) mConstructorArgs[0];
        mConstructorArgs[0] = inflaterContext;
        
        // 1. 将返回结果设置为 父 View
        View result = root;

        try {
            // 2. 寻找根布局
            int type;
            while ((type = parser.next()) != XmlPullParser.START_TAG &&
                    type != XmlPullParser.END_DOCUMENT) {
                // Empty
            }

            // 2.1 如果没有找到根布局的开始节点，抛出异常
            if (type != XmlPullParser.START_TAG) {
                throw new InflateException(parser.getPositionDescription()
                        + ": No start tag found!");
            }
            
            // 3. 得到布局起始节点名字
            final String name = parser.getName(); 
    
            ...
            
            if (TAG_MERGE.equals(name)) { // 3.1 处理节点是 Merge 的情况
                if (root == null || !attachToRoot) {
                    throw new InflateException("<merge /> can be used only with a valid "
                            + "ViewGroup root and attachToRoot=true");
                }

                rInflate(parser, root, inflaterContext, attrs, false);
            } else {
                
                // 3.2 临时根布局找到，根据名字通过反射构建 View
                final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                // 4. 声明 LayoutParams
                ViewGroup.LayoutParams params = null;

                if (root != null) {
                  
                    // 4.1 父 View 不为空时根据 父 View 构建布局的 LayoutParams
                    params = root.generateLayoutParams(attrs);
                    if (!attachToRoot) {
                        // 4.1.1 为 View 设置 LayoutParams
                        temp.setLayoutParams(params);
                    }
                }

                
                // 5. 解析该 View 下所有的子 View
                rInflateChildren(parser, temp, attrs, true);

                // 6.1 如果父 View 不为空并且设置为添加，则将布局的 View 对象添加到父 View
                if (root != null && attachToRoot) {
                    root.addView(temp, params);
                }

                // 6.2 如果 父 View 为空，则直接返回布局的 View 对象
                if (root == null || !attachToRoot) {
                    result = temp;
                }
            }

        }... // 省略 catch
        
        // 7. 返回
        return result;
    }
}  
```

接下来说的 父 View 指的是 inflate 方法的第二个参数。根 View 说的是要解析的布局文件的根 View，注意不要混淆

1. 将方法返回值设置为 父 View

2. 寻找布局的根 View 的起始节点，如果找不到，抛出异常，如果找到则继续

3. 判断根 View 的起始节点，如果是 Merge ，则解析 Merge 下的子 View。如果不是 Merge，则根据根 View 的名字构建根 View 对象

4. 如果 父 View 不为空，则根据根 View 在 xml 中声明的属性构建 LayoutParams，并为 根 View 设置该 LayoutParams，如果没有父 View 则不会为根 View 设置 LayoutParams

5. 有根 View 对象以后，再解析其内部的子 View，调用 rInflateChildren 方法，将根View 的子 View 都添加到根 View 中

6. 如果父 View 不为空，且需要添加，则将根 View 添加到父 View 中。如果父 View 为空或不需要添加，则将返回值设置为根 View

7. 返回结果


接下来我们逐个分析，先看创建 View 的方法
    
### 1. createViewFromTag 方法工作过程 ###
```Java
    // LayoutInflater
    View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
            boolean ignoreThemeAttr) {
        if (name.equals("view")) {
            name = attrs.getAttributeValue(null, "class");
        }

        ...

        try {
            View view;
            
            ... // 1. 用户可以通过设置 LayoutInflater 的 Factory 来创建 View， 默认 Factory 都为空，忽略
            
             // 2. 没有 Factory的请求，通过 onCreateView 或者 createView 创建
            if (view == null) {
                final Object lastContext = mConstructorArgs[0];
                mConstructorArgs[0] = context;
                try {
                    if (-1 == name.indexOf('.')) { // 3. View 名字中没有 . 符号，说明是系统提供的 View
                        view = onCreateView(parent, name, attrs); // 3.1 内置 View 的解析
                    } else { // 4. View 的名字中有 . 符号，说明是自定义 View
                        view = createView(name, null, attrs); // 4.1 自定义 View 的解析
                    }
                } finally {
                    mConstructorArgs[0] = lastContext;
                }
            }
            
            // 返回 View
            return view;
        } catch...
    }
```

主要步骤如下：

1. 根据 Factory 创建 View,默认忽略

2. 判断是系统 View 还是自定义 View，这里的判断方法是依据系统 View 在布局中不需要添加包名，而自定义 View 必须加包名

3. 系统 View 则调用 onCreateView 方法

4. 自定义 View 则调用 createView 方法

```Java
// PhoneLayoutInflater
@Override protected View onCreateView(String name, AttributeSet attrs) throws ClassNotFoundException {
    for (String prefix : sClassPrefixList) {
        try {
            View view = createView(name, prefix, attrs);
            if (view != null) {
                return view;
            }
        } ...
    return super.onCreateView(name, attrs);
}
```
onCreateView 方法中会先为 View 的 name 添加系统 View 的前缀来构建完整包名之后再调用 createView 方法创建 View , 也就是说不管是系统 View 还是自定义 View 最后都调用 createView 方法创建

### 2. createView 方法的工作过程 ###
```Java
// LayoutInflater
public final View createView(String name, String prefix, AttributeSet attrs)
        throws ClassNotFoundException, InflateException {
        
    // 1. 从缓存中获取构造方法并验证可用性
    Constructor<? extends View> constructor = sConstructorMap.get(name);
    if (constructor != null && !verifyClassLoader(constructor)) {
        constructor = null;
        sConstructorMap.remove(name);
    }
    
    Class<? extends View> clazz = null;

    try {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, name);

        if (constructor == null) {
            // 2. 如果缓存中没有构造方法，则通过反射获取类的 Class 对象
            clazz = mContext.getClassLoader().loadClass(
                    prefix != null ? (prefix + name) : name).asSubclass(View.class);
            
            if (mFilter != null && clazz != null) {
                boolean allowed = mFilter.onLoadClass(clazz);
                if (!allowed) {
                    failNotAllowed(name, prefix, attrs);
                }
            }
            
            // 3. 通过 Class 获取构造方法， 并添加到缓存
            constructor = clazz.getConstructor(mConstructorSignature);
            constructor.setAccessible(true);
            sConstructorMap.put(name, constructor);
        } else {
           ... 缓存中获取到构造方法的情况，省略
        }

        Object[] args = mConstructorArgs;
        args[1] = attrs;
        
        // 4. 由构造方法再通过反射机制构造该 View 类的对象，并返回
        final View view = constructor.newInstance(args);
        if (view instanceof ViewStub) {
            // Use the same context when inflating ViewStub later.
            final ViewStub viewStub = (ViewStub) view;
            viewStub.setLayoutInflater(cloneInContext((Context) args[0]));
        }
        return view;

    } catch ...
}
```

**createView 的工作过程主要就是根据 View 类的完整路径通过反射机制构造该类的对象并返回。这就是单个 View 的构造过程。如果我们的窗口是一棵视图树，LayoutInflater 需要解析完这棵树，这个任务就交给了 rInflateChildren 方法**


### 3. rInflateChildren 方法工作过程 ###

rInflateChildren 方法会直接调用 reInflate 方法
```Java
// LayoutInflater
void rInflate(XmlPullParser parser, View parent, Context context,
        AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {

    final int depth = parser.getDepth(); // 获取当前视图深度
    int type;
    
    // 挨个解析视图树
    while (((type = parser.next()) != XmlPullParser.END_TAG ||
            parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {

        if (type != XmlPullParser.START_TAG) {
            continue;
        }
        
        // 找到起始节点
        final String name = parser.getName();
        
        if (TAG_REQUEST_FOCUS.equals(name)) {
            parseRequestFocus(parser, parent);
        } else if (TAG_TAG.equals(name)) {
            parseViewTag(parser, parent, attrs);
        } else if (TAG_INCLUDE.equals(name)) { // 遇到 include 标签时
            if (parser.getDepth() == 0) { // 如果当前视图深度是 0 并且是 include 标签则抛出异常，因为 include 不能是根 View
                throw new InflateException("<include /> cannot be the root element");
            }
            parseInclude(parser, context, parent, attrs);
        } else if (TAG_MERGE.equals(name)) { // 如果遇到 Merge 则抛出异常，因为 Merge 必须是根标签
            throw new InflateException("<merge /> must be the root element");
        } else {
            // 解析普通 View 标签
            final View view = createViewFromTag(parent, name, context, attrs); // 构建 View
            final ViewGroup viewGroup = (ViewGroup) parent;
            final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs); // 根据 Parent 构建 LayoutParams
            rInflateChildren(parser, view, attrs, true); // 递归调用进行解析
            viewGroup.addView(view, params); // 将解析到的 View 添加到它的 parent 中
        }
    }

    if (finishInflate) {
        parent.onFinishInflate();
    }
}
```
**rInflate 的工作过程主要是依次解析视图树中的子 View ，并递归调用解析子 View 中的元素，并将解析到的 View 回溯回来将其添加到 父 View 中**

这样 inflate 方法的返回结果就是包含整个布局的 View 对象

**注意，这里其实有一个问题，那就是我们在使用 Fragment 和 AbsListView 的时候，如果我们在创建 View 的时候，使用了 LayoutInflater ，我们一定不可以将 inflate 的第二个参数设置为父 View ，或者说如果设置第二个参数那么第三个参数一定要设置为 false。如果设置第二个参数且第三个参数为 true，那么返回的结果就是已经添加了我们需要的 View 的父 View ，我们创建好 View 后，Fragment 和 AbsListView 会把我们返回的 View 再添加到原来的 父 View，就会抛出异常。**

### 4. LayoutInflater 的 inflate 方法工作过程总结

1. 构建解析器，由解析器获取根 View 的 name
2. 由根 View 的 name 调用 onCreateViewFromTag --> onCreateView --> createView ，createView 中通过反射构建根 View 对象
3. 获取到根 View 对象后，调用 rInflateChilder 方法，其中通过解析器循环解析根 View 中的子 View，方法依然通过 onCreateViewFromTag --> onCreateView --> createView ，createView 中通过反射构建 View 对象
4. 循环解析到根 View 的子 View 后将通过相同方法递归调用解析子 View 中的元素，并将解析到的 View 回溯回来将其添加到 父 View 中
5. 循环结束后，通过根 View 是否有父 View 为根 View 设置 xml 中设置的 LayoutParams
6. 根据参数决定是否将解析到的 View 添加到父 View
7. 返回父 View 或者根 View


## 三、由源码到实战

开发中，我们可能会有这样的需求，自定义的 ViewGroup，其子 View 可以配置一些由 ViewGroup 提供的自定义属性，例如位置、大小、样式等。这时候由于子 View 没有被重写，在渲染过程中这些配置的效果是不可能由子 View 实现的。

所以我们就可以从 ViewGroup 入手，考虑通过 ViewGroup 控制子 View 的一些效果。

#### 1. 如何拿到所有子控件里面的参数

LayoutInflater 解析 xml 时，会通过 attr 对象为 View 创建 LayoutParams 。

LayoutInflater 在解析 xml 的过程中，会调用 createViewFromTag 方法创建 View，其中会先判断 Factory 是否为空，如果 Factory 不为空会通过 Factory 创建 View。

**所以通过自定义 LayoutInflater 的 Factory ，实现其 onCreateView 方法，并调用 LayoutInflater 对象的 setFactory 方法为 LayoutInflater 对象设置自定义的 Factory，就可以自己控制 View 的创建过程。**

Factory 的 onCreateView 方法的参数是 String 类型的要创建的 View 的名字、一个上下文 Context、还有 AttributeSet 类型的 attrs 对象，attrs 中保存了 xml 中为 View 配置的属性。

使用自定义 Factory 时，我们通过构造方法中传入一个系统提供的 LayoutInflater 对象，这样在 Factory 的 onCreateView 方法中我们可以直接调用 LayoutInflater 对象的 createView 方法实现 View 对象的创建。然后在 Factory 的 onCreateView 方法中获取子 View 在 xml 中配置的属性。

使用 Factory 创建 View 时，通过 attrs 参数可以得到 xml 中 View 配置的所有属性，如果自定义 ViewGroup 为子 View 提供了自定义的属性，
在解析 attrs 时，可以通过一个类，保存 xml 中对 View 设置所有自定义的属性，然后通过 View 的 setTag 方法，为当前 View 绑定自定义的属性。

##### 2. 为每一个添加了自定义参数的子控件设置指定的效果

在 ViewGroup 中创建一个集合，遍历其子 View，将配置了自定义属性的子 View 保存在一个新的集合中。这样当自定义的 ViewGroup 的状态改变时，就可以修改集合中配置了自定义属性的 View 进行动画或特效。