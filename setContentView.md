
## 1、简介
Android开发中onCreate（）方法中一直使用setContentView（int res）方法，只是简单的了解，此方法是将我们Activity设置的布局文件添加并显示。但是从未深度的去剖析一下其原理，今天就来进入源码一探究竟。
##2.源码分析
首先我们从Acticity的oncrete（）方法出发

		     @Override
		    protected void onCreate(Bundle savedInstanceState) {
			        super.onCreate(savedInstanceState);
			        setContentView(R.layout.activity_main);
		       
		    }

非常简洁，点击setContentView（）方法，跳转到Activity的setContentView（）方法

    



		    public void setContentView(@LayoutRes int layoutResID) {
		        getWindow().setContentView(layoutResID);
				
		        initWindowDecorActionBar();
		    }

getWindow（）是什么鬼点击去


    
	public Window getWindow() {
        return mWindow;
    }

原来是返回一个Window对象。Window是一个抽象类，那他是怎么实例化的的呢，在什么地方的实例化的呢。最终在Acitity类中的attach（）方法中干的
    
	final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor) {
        attachBaseContext(context);

        mFragments.attachHost(null /*parent*/);
		//初始化Window对象
        mWindow = new PhoneWindow(this);
		//设置监听
        mWindow.setCallback(this);
			...
			...
       
    }

发现通过PhoneWindow类去实例化Window抽象类。可以得出，PhoneWindow是Window的子类，（也是唯一的一个子类），知道了GetWindow（）最终返回的是PhoneWindow实例，重新回到Activity的setContentView（）的方法，会发现其实调用的是PhoneWindow类的setContentView（）方法。PhoneWindow的方法如下：
    
	@Override
    public void setContentView(int layoutResID) {

       	//1,判断mContentParent是不是为空，为空创建，
			不为空，判断是否使用专场动画，并清除之前View

        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }
		//2、加载布局文件

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
		
        mContentParent.requestApplyInsets();

		//3.设置布局文件的加载完成的回调监听

        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
    }
	
	
其实就三个步骤，
	
1. 检查mContentParent是否为空，为空创建，不为空清除之前加载资源
2. 将布添加布局文件
3. 设置回调监听

mContentParent：最终将我们XMl文件添加到这个ViewGroup当中

那么mContentParent怎么创建的呢，点击去一探究竟。（这个方法有点长，将中间一些不是重点的用...代替）
    

	private void installDecor() {
        if (mDecor == null) {
			//创建mDecor
            mDecor = generateDecor();
            ....

        }
		
        if (mContentParent == null) {
			//创建mContentParent
            mContentParent = generateLayout(mDecor);

           ....
			....
        }
    }
	
先判断mDecor是不是为空，为空并创建它，mDecor是个什么鬼，点进去看下
    
	protected DecorView generateDecor() {
        return new DecorView(getContext(), -1);
    }	

创建了一个DecorView对象，这里大概的解释下DecorView，它是我们整个窗口的根视图，在PhoneWindow中定义为一个内部类，继承于FrameLayout，然后它有一个子view即LinearLayout，方向为竖直方向，其内有两个FrameLayout，上面的FrameLayout即为TitleBar之类的，下面的FrameLayout即为我们的ContentView，所谓的setContentView就是往这个FrameLayout里面添加我们的布局View的

现在来看下generateLayout(mDecor)

    protected ViewGroup generateLayout(DecorView decor) {
	
	//1.获取Xml中设置的属性，并根据xml中的属性设置requestFeature（）和setFlags（）
	TypedArray a = getWindowStyle();

		....
		....
	//2.根据features设置相应的布局文件
	int layoutResource;
    int features = getLocalFeatures();

			.....
			.....

	//3.设置contentParent
		
	  	View in = mLayoutInflater.inflate(layoutResource, null);
        decor.addView(in, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
        mContentRoot = (ViewGroup) in;
		
		ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
			......
			......

		return contentParent;


		}

获取到contentParent就就是里nearlayout下的Framlayout的一个子ViewGroup。到这里整个setcontView加载流程就加载完了。


