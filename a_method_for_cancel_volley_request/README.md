Volley设置网络请求生命周期的一种方法
===
> 针对既有代码进行 `Volley` 网络请求生命周期的改造

## 背景
项目组一开始做项目的时候，并没有进行`Volley`与`Activity`/`Fragment`的生命周期进行关联，即在 `Activity`/`Fragment`销毁的时候进行 `Cancel Request`操作，导致网络请求回调引发的奔溃问题频繁出现。

如何在保证当前代码尽量不改动的情况下将 `Volley` 与 `Activity`/`Fragment`进行绑定呢？

万幸的是，项目组在使用`Volley` 的时候做了一定的封装，使得所有`Request`都是通过一个叫 `RequestClient`的类进行统一请求的，而且 `Activity` 与 `Fragment`都是有基类的：）

	// 请求统一的入口
	public class RequestClient {
		
		private RequestQueue mRequestQueue;
		// ...

		public void post(String url, Response.Listener<String> listener, Response.ErrorListener errorListener) {
			// ...
		}

	}

## Volley 取消网络方法

`com.android.volley.RequestQueue`中有如下两个方法可以取消网络

	public void cancelAll(RequestQueue.RequestFilter filter) {

	}

	public void cancelAll(final Object tag) {

	}

即在请求网络的时候，将一个事先定义好的 Tag 设置到 `Recom.android.volley.Request` 中，等到要取消该网络请求（比如说界面销毁）的时候，把之前的 Tag 传入 `RequestQueue` 的`cancelAll()`即可取消该请求。

也就是说，要取消网络请求，必须要在 `Request` 中设置一个 Tag, 并且将 Tag 保存起来，等到要取消的时候使用。

## 相关理论

明白了 `Volley` 取消 `Request` 的方法后，就剩下找到一个办法去获得一个即在 `Activity`/`Fragment`中能得到又在`RequestClient`中能得到的 Tag了

### Java 内部类

如下，是一个典型的内部类例子

	package com.lwz0316.github

	// 外部类，相对于内部类而言
	public class Outter {
			
		// 内部类
		public class Inner {
			
		}

	}

很有意思的是内部类对象的 `toString()` 方法

	Outter out = new Outter();
	Outter.Inner in = out.new Inner();

	System.out.print(in.toString());

	>> com.lwz0316.github.Outter$Inner@51eef342

可以看到，内部类对象的 `toString()` 把外部类也给带出来了。

### Tag 生成

通过上面的例子，我们可以发现，通过内部类的方式，就可以生成外部类和调用内部类的方法都知道的 Tag 了。这个Tag君是让我找的好辛苦啊。

回到我们的项目中。内部类从哪里来？ 忘了我们请求的 `Listener` 回调了么？但凡是和 `Activity`/`Fragment` 相关的网络请求， `Listener` 是标配，也是导致许多 BUG 出现的地方，什么 `NullPointException`, `android.view.WindowManager$BadTokenException` 等等... 哎，在这之前好无力的感觉，不过我们现在找到 `Tag` 君了，妈妈再也不用担心我的学习，什么鬼~

`Tag` 的生成还需要对内部类对象的 `toString()` 方法进行截取，只截取 `$` 这个神奇的符号的前半部分，为啥呢？因为后面的那一部分不是固定的，外部类无法获取，外部类只知道自己的名字，这就用够了。

## 改造既有代码

### VolleyHttpLifeCircleHelper
为了 `Activity` / `Fragment` 与 `RequestClient` 调用方便，我将生成 `Tag` 与取消 `Volley` 的行为封装成了一个帮助类 `VolleyHttpLifeCircleHelper`，并提供了下面的方法

	
	/**
	 * Volley 网络请求在 {@link android.app.Activity} / {@link android.support.v4.app.Fragment} 生命周期结束时
	 * ({@link Activity#onDestroy()} / {@link Fragment#onDestroyView()}) 取消网络帮助类
	 * <p>若不需要自动取消请求，则需要在调用请求之前使用 {@link #registerWhiteUrl(String)}
	 * 方法，将url 加入白名单</P>
	 */
	public class VolleyHttpLifeCircleHelper {

		/**
	     * 将 url 加入到白名单中
	     * <p>NOTE: 请在请求前将 url 加入到白名单中</p>
	     */
	    public static void registerWhiteUrl(String url) {
			// ...
		}
	
		/**
	     * 判断 url 是否在白名单中
	     */
	    public static boolean isUrlInWhiteList(String url) {
			// ...
		}
		
		/*
		 * 获取Tag
		 * <p>如果该类不是内部类，将发生异常并直接调用String.valueOf()返回参数对象的值</p>
		 */
		public static String getTag(Object inner) {
			try {
				return getTagFromInnerInstance(inner);
			} catch(Execption e) {
				return String.valueOf(inner);
			}
		}
		
		/**
		 * 从内部类中获取 Tag
		 */
		public static String getTagFromInnerInstance(Object obj) throws Exception {
	        // may cause NullPointerException
	        String representation = obj.toString();
	        // may cause IndexOutOfBoundsException
	        return representation.substring(0, representation.indexOf("$"));
	    }
		
		/**
		 * 取消Activity / Fragment 的网络请求
		 */
		public static void cancelAllRequest(@NonNull Object object) {
	        final String tag = object.getClass().getName();
	        RequestClient.getInstance().cancelRequest(tag);
	    }
	}


### RequestClient

该类是`Volley` 的简单封装，是项目中所有网络请求的入口。

	// 请求统一的入口
	public class RequestClient {

		// ...
		
		private RequestQueue mRequestQueue;

		static {
			VolleyHttpLifeCircleHelper.register("your white url1");
			VolleyHttpLifeCircleHelper.register("your white url2");
			// ...
		}

		public void post(String url, Map<String, String> params, Response.Listener<String> listener, Response.ErrorListener errorListener) {
			// ...
			// Volley 的 Request 对象
			Request request = ...

			// 简单的逻辑，只从 listener 中取得
			String tag = VolleyHttpLifeCircleHelper.getTag(listener);

			// 向 request 中设置 tag
			request.setTag(tag);

			// 将请求放入 Volley 队列中
			mRequestQueue.add(request);
		}
		
		// 新添加 该方法，供 VolleyHttpLifeCircleHelper#cancelAllRequest() 方法 调用
		public void cancelRequest(Object tag) {
			mRequestQueue.cancelAll(new VolleyRequestFilter() {
                @NonNull
                @Override
                public Object getTag() {
                    return tag;
                }
            });
		}	
	}

上面的 `cancelRequest()` 方法中使用了一个由 `Volley` 提供的 `RequestFilter` 类，该类是为了判断 `url` 是否在白名单中，那么久不取消。否则如果 `Tag` 一致就取消请求。具体实现如下

	public abstract class VolleyRequestFilter implements RequestQueue.RequestFilter {

	    @Override
	    public boolean apply(Request<?> request) {
	        DebugLog.d("VolleyRequestFilter", "### " + this.toString());
	        final String url = request.getUrl();
			// 判断 url 是否在白名单中
	        if (VolleyHttpLifeCircleHelper.isUrlInWhiteList(url)) {
	            DebugLog.d("VolleyRequestFilter", String.format("### DO NOT cancel white request : [url -> %s]", url));
	            return false;
	        }
			// 判断 Request 的 Tag 是否与要删除的 Tag 一致
	        boolean doCancel = getTag().equals(request.getTag());
	        if (doCancel) {
	            DebugLog.d("VolleyRequestFilter", String.format("### DO cancel request : [url -> %s]", url));
	        }
	        return doCancel;
	    }
	
	    public abstract @NonNull Object getTag();
	}


### Activity & Fragment

恩，这个很简单，直接在基类的销毁方法中调用 `VolleyHttpLifeCircleHelper.cancelAllRequest(this);` 就好了, 除非你想在每个页面中都加入同样的方法:-p

- BaseActivity

		@Override
	    protected void onDestroy() {
	        VolleyHttpLifeCircleHelper.cancelAllRequest(this);
	        super.onDestroy();
	    }

- BaseFrgment

		@Override
	    public void onDestroyView() {
	        VolleyHttpLifeCircleHelper.cancelAllRequest(this);
	        super.onDestroyView();
	    }


OK，这个棘手的问题终于搞定了~
	

	