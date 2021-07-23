---
title: Android仿今日头条图片滑动退出效果
date: 2021-07-23 10:28:10
tags:
- Android
- Kotlin
categories: Android
excerpt: 使用kotlin自定义Layout，实现图片画廊和滑动退出效果
---

### Android仿今日头条图片滑动退出效果-Kotlin版

主要功能:
1. 在下滑时，随着手指的移动，图片区域跟随移动，并且activity的背景和页码逐渐变的透明
2. 滑动距离不超过设定的临界值时，会有回弹效果。
3. 滑动超过设置的临界值时，放开手指，页面滑动退出消失
4. 图片可以正常放大缩小，页面不跟随手指上下滑动
5. 使用了共享元素的页面切换效果

```kotlin
class SlideCloseLayout(context: Context, attrs: AttributeSet? = null) : FrameLayout(context, attrs) {

    private var previousX: Float = 0f
    private var previousY: Float = 0f
    private var scrollListener: LayoutScrollListener? = null

    init {
        background?.alpha = 255
    }


    override fun onInterceptTouchEvent(ev: MotionEvent?): Boolean {
        ev?.pointerCount?.let {
            if (it > 1) return false
            val y: Float = ev.rawY
            val x: Float = ev.rawX
            when (ev.action) {
                MotionEvent.ACTION_DOWN -> {
                    previousX = x
                    previousY = y
                }
                MotionEvent.ACTION_MOVE -> {
                    val diffY = y - previousY
                    val diffX = x - previousX
                    if (diffY <= 0) return false
                    if (Math.abs(diffX) + 50 < Math.abs(diffY)) {
                        return true
                    }
                }
            }
        }
        return false
    }

    override fun onTouchEvent(ev: MotionEvent?): Boolean {
        ev?.let {
            val y = ev.rawY
            val x = ev.rawX
            when (ev.action) {
                MotionEvent.ACTION_DOWN -> {
                    previousX = x
                    previousY = y
                }
                MotionEvent.ACTION_MOVE -> {
                    val diffY = Math.max(y - previousY, 0f)
                    translationY = diffY
                    val alpha = diffY / height
                    this.alpha = 1f - alpha
                }
                MotionEvent.ACTION_UP -> {
                    if (Math.abs(translationY) > (height / 4)) {
                        layoutExitAnim()
                    } else {
                        layoutRecoverAnim()
                    }
                }
                else -> {
                    return super.onTouchEvent(ev)
                }
            }
        }
        return super.onTouchEvent(ev)
    }


    fun setLayoutScrollListener(listener: LayoutScrollListener) {
        scrollListener = listener
    }

    private fun layoutRecoverAnim() {
        val recoverAnim = ObjectAnimator.ofFloat(this, "translationY", this.translationY, 0f)
        recoverAnim.duration = 100
        recoverAnim.start()
        this.alpha = 1f
    }

    private fun layoutExitAnim() {
        val exitAnim: ObjectAnimator = ObjectAnimator.ofFloat(this, "translationY", translationY, height.toFloat())
        exitAnim.addListener(object : AnimatorListenerAdapter() {
            override fun onAnimationEnd(animation: Animator?) {
                this@SlideCloseLayout.alpha = 0f
                scrollListener?.onLayoutClosed()
            }
        })
        exitAnim.addUpdateListener {
            this.alpha = 1 - translationY / height
        }
        exitAnim.duration = 200
        exitAnim.start()
    }
}

interface LayoutScrollListener {
    fun onLayoutClosed()
}
```

这里自定义了一个Layout，为了能够监听手指滑动的事件

布局文件只要在这个自定义下嵌套即可

```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android">
 
 
    <jp.hotpepper.android.beauty.hair.application.widget.SlideCloseLayout
        android:id="@+id/slide_close_layout"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:background="@color/beauty_text_black">
 
        <android.support.v4.view.ViewPager
            android:id="@+id/coupon_photo_view_pager"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_gravity="center" />
 
        <TextView
            android:id="@+id/text_view_current_page"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_gravity="bottom|center"
            android:layout_marginBottom="20dp"
            android:textColor="@color/beauty_text_white" />
 
        <ImageView
            android:id="@+id/image_view_button_close"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_gravity="top|left"
            android:layout_marginLeft="15dp"
            android:layout_marginTop="25dp"
            android:background="@drawable/btn_close_circle"
            android:foreground="?android:attr/selectableItemBackground" />
 
    </jp.hotpepper.android.beauty.hair.application.widget.SlideCloseLayout>
</layout>
```

在这里我们因为使用了自动绑定，所以最外层需要绑定一个Layout

然后是我们的图片画廊展示类

```kotlin
class CouponPhotoViewPagerActivity : BaseActivity() {
 
    private val photoUrls: ArrayList<String> by extra()
 
    private val binding by binding<ActivityCouponPhotoViewPagerBinding>(R.layout.activity_coupon_photo_view_pager)
 
    private val viewPager: ViewPager by lazy {
        binding.couponPhotoViewPager
    }
 
    private val closeImageView: ImageView by lazy {
        binding.imageViewButtonClose
    }
 
    private val showCurrentPage: TextView by lazy {
        binding.textViewCurrentPage
    }
 
    private val slideCloseLayout: SlideCloseLayout by lazy {
        binding.slideCloseLayout
    }
 
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        initComponent()
    }
    
    private fun initComponent() {
        activityComponent.inject(this)
        viewPager.adapter = CouponPhotoViewPagerAdapter(photoUrls)
        viewPager.setTransitionNameCompat(SHARED_ELEMENT_NAME)
        showCurrentPage.text = getString(R.string.coupon_photo_view_current_page, 1, photoUrls.size)
        initEventListener()
    }
    
    private fun initEventListener() {
        closeImageView.setOnClickListener {
            finishAfterTransitionCompat()
        }
        viewPager.addOnPageChangeListener(object : ViewPager.SimpleOnPageChangeListener() {
            override fun onPageSelected(p0: Int) {
                showCurrentPage.text = getString(R.string.coupon_photo_view_current_page, p0 + 1, photoUrls.size)
            }
        })

        slideCloseLayout.setLayoutScrollListener(object : LayoutScrollListener {
            override fun onLayoutClosed() {
                finish()
                overridePendingTransition(R.anim.fade_in, R.anim.fade_out)
            }
        })
    }
    
    companion object {
        private const val SHARED_ELEMENT_NAME = "sharedView"
        fun intent(context: Context, photoUrls: List<String>): Intent =
                Intent(context, CouponPhotoViewPagerActivity::class.java)
                        .put(CouponPhotoViewPagerActivity::photoUrls, ArrayList(photoUrls))

        fun transitionOptions(activity: Activity, sharedElement: View) = ActivityOptionsCompat.makeSceneTransitionAnimation(activity, sharedElement, SHARED_ELEMENT_NAME).toBundle()
    }
}
```

给这个activity定一个主题，不然在下滑的时候是不能透明的，也看不到之前的activity

```xml
<style name="SlideCloseTheme">
        <item name="windowNoTitle">true</item>
        <item name="android:windowFullscreen">?android:windowNoTitle</item>
        <item name="android:windowIsTranslucent">true</item>
        <item name="android:windowBackground">@android:color/transparent</item>
        <item name="android:windowNoTitle">true</item>
</style>
```
这个主题表示activity是全屏显示并且可以透明化的。

然后，在manifest.xml中应用这个主题

```xml

<activity
            android:name=".application.activity.CouponPhotoViewPagerActivity"
            android:configChanges="orientation"
            android:screenOrientation="portrait"
            android:theme="@style/SlideCloseTheme" />
```

实现效果还是很不错的。
