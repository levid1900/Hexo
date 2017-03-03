---
title: 关于ListView的单选多选问题
date: 2016-03-28 16:43:23
tags: Android
---
在开发项目中，经常会遇到一种需求，在ListView中的单选或者多选模式.

通常情况下，我们都是这样做的：在Adapter记录一个数组或者Map对象，对每一个Item的选择状态进行记录。然后在ListView进行点击事件后，根据记录中的item状态再刷新每一个Item。

这样做比较繁琐，如果需求只是很简单的单选界面，需要额外写非常多的代码，维护起来比较困难，而且容易遇到Bug.

# ListView的ChoiceMode
ListView有个方法叫[setChoiceMode()](http://developer.android.com/intl/zh-cn/reference/android/widget/AbsListView.html#setChoiceMode(int)

```bash
public void setChoiceMode (int choiceMode)

Added in API level 1
Defines the choice behavior for the List. By default, Lists do not have any choice behavior (CHOICE_MODE_NONE). By setting the choiceMode to CHOICE_MODE_SINGLE, the List allows up to one item to be in a chosen state. By setting the choiceMode to CHOICE_MODE_MULTIPLE, the list allows any number of items to be chosen.

Parameters
choiceMode	int: One of CHOICE_MODE_NONE, CHOICE_MODE_SINGLE, or CHOICE_MODE_MULTIPLE#
```
使用这个方法可以轻易的创建出一个单选或者多选模式的ListView.

**Showing Code**

```java
String[] data = new String[]{
               "Item1",
               "Item2",
               "Item3",
               "Item4",
               "Item5",
               "Item6",
               "Item7",
       };
mListView = (ListView)findViewById(R.id.listview);
mListView.setAdapter(new ArrayAdapter<>(this, android.R.layout.simple_list_item_single_choice, data));
mListView.setChoiceMode(ListView.CHOICE_MODE_SINGLE);
```
要获取被选择的Item,使用**mListView.getCheckedItemPosition()**。多选模式下，使用**mListView.getCheckedItemPositions()**

**效果图**

<img src="http://7xsdgx.com1.z0.glb.clouddn.com/20160328-180253.png?imageView2/2/w/200" width="200" height="350"/>

使用这种模式，必须使用**android.R.layout.simple_list_item_single_choice**这个布局，来看一下这个布局到底是什么样的

```java
<CheckedTextView xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@android:id/text1"
    android:layout_width="match_parent"
    android:layout_height="?android:attr/listPreferredItemHeightSmall"
    android:textAppearance="?android:attr/textAppearanceListItemSmall"
    android:gravity="center_vertical"
    android:checkMark="?android:attr/listChoiceIndicatorSingle"
    android:paddingStart="?android:attr/listPreferredItemPaddingStart"
    android:paddingEnd="?android:attr/listPreferredItemPaddingEnd" />
```

可以看到这个Layout里面使用了一个CheckedTextView的自定义布局，继续看看它什么样子

```java
/**
 * An extension to TextView that supports the {@link android.widget.Checkable} interface.
 * This is useful when used in a {@link android.widget.ListView ListView} where the it's
 * {@link android.widget.ListView#setChoiceMode(int) setChoiceMode} has been set to
 * something other than {@link android.widget.ListView#CHOICE_MODE_NONE CHOICE_MODE_NONE}.
 *
 * @attr ref android.R.styleable#CheckedTextView_checked
 * @attr ref android.R.styleable#CheckedTextView_checkMark
 */
public class CheckedTextView extends TextView implements Checkable {

    public CheckedTextView(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        super(context, attrs, defStyleAttr, defStyleRes);

        final TypedArray a = context.obtainStyledAttributes(
                attrs, R.styleable.CheckedTextView, defStyleAttr, defStyleRes);

        final Drawable d = a.getDrawable(R.styleable.CheckedTextView_checkMark);
        if (d != null) {
            setCheckMarkDrawable(d);
        }

        if (a.hasValue(R.styleable.CheckedTextView_checkMarkTintMode)) {
            mCheckMarkTintMode = Drawable.parseTintMode(a.getInt(
                    R.styleable.CheckedTextView_checkMarkTintMode, -1), mCheckMarkTintMode);
            mHasCheckMarkTintMode = true;
        }

        if (a.hasValue(R.styleable.CheckedTextView_checkMarkTint)) {
            mCheckMarkTintList = a.getColorStateList(R.styleable.CheckedTextView_checkMarkTint);
            mHasCheckMarkTint = true;
        }

        mCheckMarkGravity = a.getInt(R.styleable.CheckedTextView_checkMarkGravity, Gravity.END);

        final boolean checked = a.getBoolean(R.styleable.CheckedTextView_checked, false);
        setChecked(checked);

        a.recycle();

        applyCheckMarkTint();
    }

    public void toggle() {
        setChecked(!mChecked);
    }

    @ViewDebug.ExportedProperty
    public boolean isChecked() {
        return mChecked;
    }

    /**
     * <p>Changes the checked state of this text view.</p>
     *
     * @param checked true to check the text, false to uncheck it
     */
    public void setChecked(boolean checked) {
        if (mChecked != checked) {
            mChecked = checked;
            refreshDrawableState();
            notifyViewAccessibilityStateChangedIfNeeded(
                    AccessibilityEvent.CONTENT_CHANGE_TYPE_UNDEFINED);
        }
    }

    ...
}
```

代码太长，有兴趣的可以看[这里:CheckedTextView](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.0.0_r1/android/widget/CheckedTextView.java#CheckedTextView)

可以看到，CheckedTextView是一个自定义TextView, 实现了Checkable接口，外部可以定义两个属性**CheckedTextView_checked**和**CheckedTextView_checkMark**来定义Item是否被选中以及CheckBox的外观。

以上布局比较简单，只是一个TextView来显示而已。如果项目中需求比较复杂，需要自定义布局，是否有办法搞定？

# 自定义布局
参考上面的CheckedTextView,来实现一个自定义布局的Item.
## 自定义View的实现

```java
public class SingleChoiceItem extends RelativeLayout implements Checkable {
    private CheckBox mCheckBox;

    public SingleChoiceItem(Context context) {
        super(context);
    }

    public SingleChoiceItem(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    public SingleChoiceItem(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }

    @Override
    protected void onFinishInflate() {
        super.onFinishInflate();
        for(int i = 0; i < getChildCount(); i++){
            View view = getChildAt(i);
            if(view instanceof CheckBox){
                mCheckBox = (CheckBox)view;
                mCheckBox.setClickable(false);
                mCheckBox.setFocusable(false);
                mCheckBox.setFocusableInTouchMode(false);
                break;
            }
        }

    }

    @Override
    public void setChecked(boolean checked) {
        if(mCheckBox != null){
            mCheckBox.setChecked(checked);
        }
    }

    @Override
    public boolean isChecked() {
        if(mCheckBox != null){
            return mCheckBox.isChecked();
        }
        return false;
    }

    @Override
    public void toggle() {
        if(mCheckBox != null){
            setChecked(!isChecked());
        }
    }
}
```
新建一个SingleChoiceItem类，实现Checkable接口。在onFinishInflate()方法中找到子元素中的Checkbox，然后分别实现setChecked()、isChecked()以及toggle()。

**注意：** 需要屏蔽Checkbox的点击事件以及焦点事件，否则父类即ListView拿不到点击事件。

## 实现XML布局
然后新建一个Layout:item_listview

```java
<?xml version="1.0" encoding="utf-8"?>
<com.testapplication.singleList.SingleChoiceItem xmlns:android="http://schemas.android.com/apk/res/android"
              android:orientation="vertical"
              android:layout_width="match_parent"
              android:layout_height="match_parent"
              android:gravity="center_vertical"
              android:minHeight="60dp">

    <LinearLayout
        android:layout_width="wrap_content"
        android:layout_height="match_parent"
        android:orientation="vertical">

        <TextView
            android:id="@+id/item_listview_title_tv"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"/>

        <TextView
            android:id="@+id/item_listview_sub_tv"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            />
    </LinearLayout>


    <CheckBox
        android:id="@+id/item_listview_checkbox"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:clickable="false"
        android:focusable="false"
        android:focusableInTouchMode="false"
        android:layout_alignParentRight="true"/>

</com.testapplication.singleList.SingleChoiceItem>
```
在layout中可以自定义内容，只需要包括一个Checkbox即可。

## JAVA代码实现
最后是实现代码。

```java
public class ListTestActivity extends AppCompatActivity {

    private ListView mListView;
    String[] mData = new String[]{
            "Item1",
            "Item2",
            "Item3",
            "Item4",
            "Item5",
            "Item6",
            "Item7",
    };

    String[] mContent = new String[]{
            "subContent1",
            "subContent2",
            "subContent3",
            "subContent4",
            "subContent5",
            "subContent6",
            "subContent7",
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_list_test);
        Toolbar toolbar = (Toolbar) findViewById(R.id.toolbar);
        setSupportActionBar(toolbar);

        mListView = (ListView)findViewById(R.id.listview);
        mListView.setAdapter(new MyListAdapter());
        mListView.setChoiceMode(ListView.CHOICE_MODE_SINGLE);
        mListView.setOnItemClickListener(new AdapterView.OnItemClickListener() {
            @Override
            public void onItemClick(AdapterView<?> parent, View view, int position, long id) {
                Toast.makeText(ListTestActivity.this, String.valueOf(mListView.getCheckedItemPosition()), Toast.LENGTH_LONG).show();
            }
        });
    }

    class MyListAdapter extends BaseAdapter{

        @Override
        public int getCount() {
            return mData.length;
        }

        @Override
        public Object getItem(int position) {
            return mData[position];
        }

        @Override
        public long getItemId(int position) {
            return 0;
        }

        @Override
        public View getView(int position, View convertView, ViewGroup parent) {
            SingleChoiceItem view;
            if(convertView == null){
                view = (SingleChoiceItem)LayoutInflater.from(ListTestActivity.this).inflate(R.layout.item_listview, null);

                TextView title = (TextView)view.findViewById(R.id.item_listview_title_tv);
                TextView content = (TextView)view.findViewById(R.id.item_listview_sub_tv);
                title.setText(mData[position]);
                content.setText(mContent[position]);
            }else{
                view = (SingleChoiceItem)convertView;
            }
            return view;
        }
    }

}
```

来看下效果图

<img src="http://7xsdgx.com1.z0.glb.clouddn.com/20160328-191138.png?imageView2/2/w/200" width="200" />
