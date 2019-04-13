# Google-AAD

考纲：https://developers.google.com/training/certification/associate-android-developer/study-guide/

## Android Core:

### Understand the architecture of the Android system

#### [Android Basics](https://developer.android.com/guide/components/fundamentals)

#### Four app components:

* [Activities](https://developer.android.com/guide/components/activities/intro-activities)

Activity life cycle:

![lifecycle](https://developer.android.com/guide/components/images/activity_lifecycle.png)

Fragment life cycle:

![](https://developer.android.com/images/fragment_lifecycle.png)

* [Services](https://developer.android.com/reference/android/app/Service)

A Service itself is actually very simple, providing two main features:

A facility for the application to tell the system about something it wants to be doing in the background (even when the user is not directly interacting with the application). This corresponds to calls to `Context.startService()`, which ask the system to schedule work for the service, to be run until the service or someone else explicitly stop it.

A facility for an application to expose some of its functionality to other applications. This corresponds to calls to` Context.bindService()`, which allows a long-standing connection to be made to the service in order to interact with it.

* [BroadcastReveivers](https://developer.android.com/guide/components/broadcasts#system-broadcasts)

* [ContentProviders](https://developer.android.com/guide/topics/providers/content-providers)

[Room with content provider example](https://github.com/googlesamples/android-architecture-components/tree/master/PersistenceContentProviderSample)

```xml
<provider
    android:name=".provider.SampleContentProvider"
    android:authorities="com.example.android.contentprovidersample.provider"
    android:exported="true"
    android:permission="com.example.android.contentprovidersample.provider.READ_WRITE"/>
```
```java
/**
 * A {@link ContentProvider} based on a Room database.
 *
 * <p>Note that you don't need to implement a ContentProvider unless you want to expose the data
 * outside your process or your application already uses a ContentProvider.</p>
 */
public class SampleContentProvider extends ContentProvider {

    /** The authority of this content provider. */
    public static final String AUTHORITY = "com.example.android.contentprovidersample.provider";

    /** The URI for the Cheese table. */
    public static final Uri URI_CHEESE = Uri.parse(
            "content://" + AUTHORITY + "/" + Cheese.TABLE_NAME);

    /** The match code for some items in the Cheese table. */
    private static final int CODE_CHEESE_DIR = 1;

    /** The match code for an item in the Cheese table. */
    private static final int CODE_CHEESE_ITEM = 2;

    /** The URI matcher. */
    private static final UriMatcher MATCHER = new UriMatcher(UriMatcher.NO_MATCH);

    static {
        MATCHER.addURI(AUTHORITY, Cheese.TABLE_NAME, CODE_CHEESE_DIR);
        MATCHER.addURI(AUTHORITY, Cheese.TABLE_NAME + "/*", CODE_CHEESE_ITEM);
    }

    @Override
    public boolean onCreate() {
        return true;
    }

    @Nullable
    @Override
    public Cursor query(@NonNull Uri uri, @Nullable String[] projection, @Nullable String selection,
            @Nullable String[] selectionArgs, @Nullable String sortOrder) {
        final int code = MATCHER.match(uri);
        if (code == CODE_CHEESE_DIR || code == CODE_CHEESE_ITEM) {
            final Context context = getContext();
            if (context == null) {
                return null;
            }
            CheeseDao cheese = SampleDatabase.getInstance(context).cheese();
            final Cursor cursor;
            if (code == CODE_CHEESE_DIR) {
                cursor = cheese.selectAll();
            } else {
                cursor = cheese.selectById(ContentUris.parseId(uri));
            }
            cursor.setNotificationUri(context.getContentResolver(), uri);
            return cursor;
        } else {
            throw new IllegalArgumentException("Unknown URI: " + uri);
        }
    }

    @Nullable
    @Override
    public String getType(@NonNull Uri uri) {
        switch (MATCHER.match(uri)) {
            case CODE_CHEESE_DIR:
                return "vnd.android.cursor.dir/" + AUTHORITY + "." + Cheese.TABLE_NAME;
            case CODE_CHEESE_ITEM:
                return "vnd.android.cursor.item/" + AUTHORITY + "." + Cheese.TABLE_NAME;
            default:
                throw new IllegalArgumentException("Unknown URI: " + uri);
        }
    }

    @Nullable
    @Override
    public Uri insert(@NonNull Uri uri, @Nullable ContentValues values) {
        switch (MATCHER.match(uri)) {
            case CODE_CHEESE_DIR:
                final Context context = getContext();
                if (context == null) {
                    return null;
                }
                final long id = SampleDatabase.getInstance(context).cheese()
                        .insert(Cheese.fromContentValues(values));
                context.getContentResolver().notifyChange(uri, null);
                return ContentUris.withAppendedId(uri, id);
            case CODE_CHEESE_ITEM:
                throw new IllegalArgumentException("Invalid URI, cannot insert with ID: " + uri);
            default:
                throw new IllegalArgumentException("Unknown URI: " + uri);
        }
    }

    @Override
    public int delete(@NonNull Uri uri, @Nullable String selection,
            @Nullable String[] selectionArgs) {
        switch (MATCHER.match(uri)) {
            case CODE_CHEESE_DIR:
                throw new IllegalArgumentException("Invalid URI, cannot update without ID" + uri);
            case CODE_CHEESE_ITEM:
                final Context context = getContext();
                if (context == null) {
                    return 0;
                }
                final int count = SampleDatabase.getInstance(context).cheese()
                        .deleteById(ContentUris.parseId(uri));
                context.getContentResolver().notifyChange(uri, null);
                return count;
            default:
                throw new IllegalArgumentException("Unknown URI: " + uri);
        }
    }

    @Override
    public int update(@NonNull Uri uri, @Nullable ContentValues values, @Nullable String selection,
            @Nullable String[] selectionArgs) {
        switch (MATCHER.match(uri)) {
            case CODE_CHEESE_DIR:
                throw new IllegalArgumentException("Invalid URI, cannot update without ID" + uri);
            case CODE_CHEESE_ITEM:
                final Context context = getContext();
                if (context == null) {
                    return 0;
                }
                final Cheese cheese = Cheese.fromContentValues(values);
                cheese.id = ContentUris.parseId(uri);
                final int count = SampleDatabase.getInstance(context).cheese()
                        .update(cheese);
                context.getContentResolver().notifyChange(uri, null);
                return count;
            default:
                throw new IllegalArgumentException("Unknown URI: " + uri);
        }
    }

    @NonNull
    @Override
    public ContentProviderResult[] applyBatch(
            @NonNull ArrayList<ContentProviderOperation> operations)
            throws OperationApplicationException {
        final Context context = getContext();
        if (context == null) {
            return new ContentProviderResult[0];
        }
        final SampleDatabase database = SampleDatabase.getInstance(context);
        database.beginTransaction();
        try {
            final ContentProviderResult[] result = super.applyBatch(operations);
            database.setTransactionSuccessful();
            return result;
        } finally {
            database.endTransaction();
        }
    }

    @Override
    public int bulkInsert(@NonNull Uri uri, @NonNull ContentValues[] valuesArray) {
        switch (MATCHER.match(uri)) {
            case CODE_CHEESE_DIR:
                final Context context = getContext();
                if (context == null) {
                    return 0;
                }
                final SampleDatabase database = SampleDatabase.getInstance(context);
                final Cheese[] cheeses = new Cheese[valuesArray.length];
                for (int i = 0; i < valuesArray.length; i++) {
                    cheeses[i] = Cheese.fromContentValues(valuesArray[i]);
                }
                return database.cheese().insertAll(cheeses).length;
            case CODE_CHEESE_ITEM:
                throw new IllegalArgumentException("Invalid URI, cannot insert with ID: " + uri);
            default:
                throw new IllegalArgumentException("Unknown URI: " + uri);
        }
    }

}
```

Unlike activities, services, and broadcast receivers, content providers are not activated by intents. Rather, they are activated when targeted by a request from a ContentResolver.

### Be able to describe the basic building blocks of an Android app

* [App components](https://developer.android.com/guide/components/fundamentals#Components)

* [Manifest.xml](https://developer.android.com/guide/components/fundamentals#Manifest)

* [App resources](https://developer.android.com/guide/components/fundamentals#Resources)

### [Know how to build and run an Android app](https://developer.android.com/training/basics/firstapp/creating-project)

### Display simple messages in a popup using a Toast or a Snackbar

#### [Toast](https://developer.android.com/guide/topics/ui/notifiers/toasts):

```java
Context context = getApplicationContext();
CharSequence text = "Hello toast!";
int duration = Toast.LENGTH_SHORT;

Toast toast = Toast.makeText(context, text, duration);
toast.show();
```

#### [Snackbar](https://developer.android.com/reference/android/support/design/widget/Snackbar):

```java
Snackbar mySnackbar = Snackbar.make(findViewById(R.id.myCoordinatorLayout),
                                R.string.email_archived, Snackbar.LENGTH_SHORT);
mySnackbar.setAction(R.string.undo_string, new MyUndoListener());
mySnackbar.show();
```



### [Be able to display a message outside your app's UI using Notifications](https://developer.android.com/guide/topics/ui/notifiers/notifications)

```
dependencies {
    implementation "com.android.support:support-compat:28.0.0"
}
```

#### Create Channel:

```java
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
    CharSequence name = getString(R.string.channel_name);
    String description = getString(R.string.channel_description);
    int importance = NotificationManager.IMPORTANCE_DEFAULT;
    NotificationChannel channel = new NotificationChannel(CHANNEL_ID, name, importance);
    channel.setDescription(description);
    // Register the channel with the system; you can't change the importance
    // or other notification behaviors after this
    NotificationManager notificationManager = getSystemService(NotificationManager.class);
    notificationManager.createNotificationChannel(channel);
}
```

#### Builder:

```java
NotificationCompat.Builder builder = new NotificationCompat.Builder(this, CHANNEL_ID)
        .setSmallIcon(R.drawable.notification_icon)
        .setContentTitle(textTitle)
        .setContentText(textContent)
        .setPriority(NotificationCompat.PRIORITY_DEFAULT);
```
#### Show:

```java
NotificationManagerCompat notificationManager = NotificationManagerCompat.from(this);

// notificationId is a unique int for each notification that you must define
notificationManager.notify(notificationId, builder.build());
```


### Understand how to localize an app

#### [Doc](https://developer.android.com/guide/topics/resources/localization)

#### [Codelab](https://codelabs.developers.google.com/codelabs/advanced-android-training-use-locale/index.html#0)

### Be able to schedule a background task using JobScheduler

#### [CodeLab](https://codelabs.developers.google.com/codelabs/android-training-job-scheduler/index.html)

```xml
<service
    android:name=".NotificationJobService"
    android:enabled="true"
    android:exported="true"
    android:permission="android.permission.BIND_JOB_SERVICE" />
```

JobService:

```java
public class NotificationJobService extends JobService {

    @Override
    public boolean onStartJob(JobParameters params) {
        //Create the notification channel
        createNotificationChannel();
        NotificationTask task=new NotificationTask();
        task.doInBackground();
        return true;//return false because the job is finished here
    }


    @Override
    public boolean onStopJob(JobParameters params) {
        return true;//if job fails, reschedule the job instead of drop
    }

}

```

JobScheduler:

```java
private JobScheduler mScheduler;
private static final int JOB_ID = 0;
public void scheduleJob(){
        int selectedNetworkOption=JobInfo.NETWORK_TYPE_NONE;
        mScheduler=(JobScheduler)getSystemService(JOB_SCHEDULER_SERVICE);
        ComponentName serviceName = new ComponentName(getPackageName(),
        NotificationJobService.class.getName());
        JobInfo.Builder builder = new JobInfo.Builder(JOB_ID, serviceName);
        builder.setRequiredNetworkType(selectedNetworkOption);
        builder.setRequiresDeviceIdle(mDeviceIdleSwitch.isChecked());
        builder.setRequiresCharging(mDeviceChargingSwitch.isChecked());
        builder.setOverrideDeadline(seekBarInteger * 1000);//过了ddl就不run你的job了
        //Schedule the job and notify the user
        JobInfo myJobInfo = builder.build();
mScheduler.schedule(myJobInfo);
}

public void cancelJobs(View view) {
if (mScheduler!=null){
    mScheduler.cancelAll();
    mScheduler = null;
    Toast.makeText(this, "Jobs cancelled", Toast.LENGTH_SHORT).show();
}
}
```

## User interface:

### Understand the Android activity lifecycle

### Be able to create an Activity that displays a Layout

### [Be able to construct a UI with ConstraintLayout](https://developer.android.com/training/constraint-layout/)

### [Understand how to create a custom View class and add it to a Layout](https://developer.android.com/guide/topics/ui/custom-components)

[CodeLab](https://codelabs.developers.google.com/codelabs/advanced-android-training-custom-view-from-scratch/index.html)

```java
public class DialView extends View {

    private static int SELECTION_COUNT = 4; // Total number of selections.
    private float mWidth;                   // Custom view width.
    private float mHeight;                  // Custom view height.
    private Paint mTextPaint;               // For text in the view.
    private Paint mDialPaint;               // For dial circle in the view.
    private float mRadius;                  // Radius of the circle.
    private int mActiveSelection;           // The active selection.
    // String buffer for dial labels and float for ComputeXY result.
    private final StringBuffer mTempLabel = new StringBuffer(8);
    private final float[] mTempResult = new float[2];
    public DialView(Context context) {
        super(context);
        init();
    }

    public DialView(Context context,  @Nullable AttributeSet attrs) {
        super(context, attrs);
        init();
    }

    public DialView(Context context,  @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        init();
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        // Draw the dial.
        canvas.drawCircle(mWidth / 2, mHeight / 2, mRadius, mDialPaint);
        // Draw the text labels.
        final float labelRadius = mRadius + 20;
        StringBuffer label = mTempLabel;
        for (int i = 0; i < SELECTION_COUNT; i++) {
            float[] xyData = computeXYForPosition(i, labelRadius);
            float x = xyData[0];
            float y = xyData[1];
            label.setLength(0);
            label.append(i);
            canvas.drawText(label, 0, label.length(), x, y, mTextPaint);
        }
        // Draw the indicator mark.
        final float markerRadius = mRadius - 35;
        float[] xyData = computeXYForPosition(mActiveSelection,
                markerRadius);
        float x = xyData[0];
        float y = xyData[1];
        canvas.drawCircle(x, y, 20, mTextPaint);
    }

    @Override
    protected void onSizeChanged(int w, int h, int oldw, int oldh) {
        // Calculate the radius from the width and height.
        mWidth = w;
        mHeight = h;
        mRadius = (float) (Math.min(mWidth, mHeight) / 2 * 0.8);
    }
}
```

Custom Attribute:

attrs.xml
```xml

<resources>
    <declare-styleable name="DialView">
        <attr name="fanOnColor" format="reference|color" />
        <attr name="fanOffColor" format="reference|color" />
    </declare-styleable>
</resources>
```
get Attributes
```java
// Get the custom attributes (fanOnColor and fanOffColor) if available.
if (attrs! = null) {
    TypedArray typedArray = getContext().obtainStyledAttributes(attrs,
                            R.styleable.DialView,
                            0, 0);
    // Set the fan on and fan off colors from the attribute values.
    mFanOnColor = typedArray.getColor(R.styleable.DialView_fanOnColor,
                    mFanOnColor);
    mFanOffColor = typedArray.getColor(R.styleable.DialView_fanOffColor,
                    mFanOffColor);
    typedArray.recycle();
```




### [Know how to implement a custom app theme](https://developer.android.com/guide/topics/ui/look-and-feel/themes)



[CodeLab](https://codelabs.developers.google.com/codelabs/android-training-drawables-styles-and-themes/index.html)

### [Be able to add accessibility hooks to a custom View](https://developer.android.com/guide/topics/ui/accessibility/custom-views)

```java
@Override
public boolean onKeyUp (int keyCode, KeyEvent event) {
    if (keyCode == KeyEvent.KEYCODE_DPAD_LEFT) {
        currentValue--;
        sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_TEXT_CHANGED);
        return true;
    }
    ...
}

```

### Know how to apply content descriptions to views for accessibility

关爱残障人士从我做起

### [Understand how to display items in a RecyclerView](https://developer.android.com/guide/topics/ui/layout/recyclerview)

Activity:

```java
public class MyActivity extends Activity {
    private RecyclerView recyclerView;
    private RecyclerView.Adapter mAdapter;
    private RecyclerView.LayoutManager layoutManager;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.my_activity);
        recyclerView = (RecyclerView) findViewById(R.id.my_recycler_view);

        // use this setting to improve performance if you know that changes
        // in content do not change the layout size of the RecyclerView
        recyclerView.setHasFixedSize(true);

        // use a linear layout manager
        layoutManager = new LinearLayoutManager(this);
        recyclerView.setLayoutManager(layoutManager);

        // specify an adapter (see also next example)
        mAdapter = new MyAdapter(myDataset);
        recyclerView.setAdapter(mAdapter);
    }
    // ...
}
```

Adapter.java:

```java

public class MyAdapter extends RecyclerView.Adapter<MyAdapter.MyViewHolder> {
    private String[] mDataset;

    // Provide a reference to the views for each data item
    // Complex data items may need more than one view per item, and
    // you provide access to all the views for a data item in a view holder
    public static class MyViewHolder extends RecyclerView.ViewHolder {
        // each data item is just a string in this case
        public TextView textView;
        public MyViewHolder(TextView v) {
            super(v);
            textView = v;
        }
    }

    // Provide a suitable constructor (depends on the kind of dataset)
    public MyAdapter(String[] myDataset) {
        mDataset = myDataset;
    }

    // Create new views (invoked by the layout manager)
    @Override
    public MyAdapter.MyViewHolder onCreateViewHolder(ViewGroup parent,
                                                   int viewType) {
        // create a new view
        TextView v = (TextView) LayoutInflater.from(parent.getContext())
                .inflate(R.layout.my_text_view, parent, false);
        ...
        MyViewHolder vh = new MyViewHolder(v);
        return vh;
    }

    // Replace the contents of a view (invoked by the layout manager)
    @Override
    public void onBindViewHolder(MyViewHolder holder, int position) {
        // - get element from your dataset at this position
        // - replace the contents of the view with that element
        holder.textView.setText(mDataset[position]);

    }

    // Return the size of your dataset (invoked by the layout manager)
    @Override
    public int getItemCount() {
        return mDataset.length;
    }
}

```




### Be able to bind local data to a RecyclerView list using the Paging library

![](https://codelabs.developers.google.com/codelabs/android-paging/img/511a702ae4af43cd.png)

```java
@Dao
public interface ConcertDao {
    // The Integer type parameter tells Room to use a PositionalDataSource
    // object, with position-based loading under the hood.
    @Query("SELECT * FROM concerts ORDER BY date DESC")
    DataSource.Factory<Integer, Concert> concertsByDate();
}

public class ConcertViewModel extends ViewModel {
    private ConcertDao concertDao;
    public final LiveData<PagedList<Concert>> concertList;

    public ConcertViewModel(ConcertDao concertDao) {
        this.concertDao = concertDao;
        concertList = new LivePagedListBuilder<>(
            concertDao.concertsByDate(), /* page size */ 50).build();
    }
}

public class ConcertActivity extends AppCompatActivity {
    @Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ConcertViewModel viewModel =
                ViewModelProviders.of(this).get(ConcertViewModel.class);
        RecyclerView recyclerView = findViewById(R.id.concert_list);
        ConcertAdapter adapter = new ConcertAdapter();
        viewModel.concertList.observe(this, adapter::submitList);
        recyclerView.setAdapter(adapter);
    }
}

public class ConcertAdapter
        extends PagedListAdapter<Concert, ConcertViewHolder> {
    protected ConcertAdapter() {
        super(DIFF_CALLBACK);
    }

    @Override
    public void onBindViewHolder(@NonNull ConcertViewHolder holder,
            int position) {
        Concert concert = getItem(position);
        if (concert != null) {
            holder.bindTo(concert);
        } else {
            // Null defines a placeholder item - PagedListAdapter automatically
            // invalidates this row when the actual object is loaded from the
            // database.
            holder.clear();
        }
    }

    private static DiffUtil.ItemCallback<Concert> DIFF_CALLBACK =
            new DiffUtil.ItemCallback<Concert>() {
        // Concert details may have changed if reloaded from the database,
        // but ID is fixed.
        @Override
        public boolean areItemsTheSame(Concert oldConcert, Concert newConcert) {
            return oldConcert.getId() == newConcert.getId();
        }

        @Override
        public boolean areContentsTheSame(Concert oldConcert,
                Concert newConcert) {
            return oldConcert.equals(newConcert);
        }
    };
}
```

If the source is from Network or other sources, you should create and inherent the Datasource.class and Datasource.Factory.class yoruself


### [Know how to implement menu-based navigation](https://developer.android.com/guide/topics/ui/menus)

Menu item:
```xml
<?xml version="1.0" encoding="utf-8"?>
<menu xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:id="@+id/file"
          android:title="@string/file" >
        <!-- "file" submenu -->
        <menu>
            <item android:id="@+id/create_new"
                  android:title="@string/create_new" />
            <item android:id="@+id/open"
                  android:title="@string/open" />
        </menu>
    </item>
</menu>
```

Activity:

```java
@Override
public boolean onCreateOptionsMenu(Menu menu) {
    MenuInflater inflater = getMenuInflater();
    inflater.inflate(R.menu.game_menu, menu);
    return true;
}

@Override
public boolean onOptionsItemSelected(MenuItem item) {
    // Handle item selection
    switch (item.getItemId()) {
        case R.id.new_game:
            newGame();
            return true;
        case R.id.help:
            showHelp();
            return true;
        default:
            return super.onOptionsItemSelected(item);
    }
}

```



### [Understand how to implement drawer navigation](https://developer.android.com/training/implementing-navigation/nav-drawer)

```
dependencies {
  implementation 'com.android.support:appcompat-v7:28.0.0'
  implementation 'com.android.support:design:28.0.0'
}
```

layout:

```xml
<?xml version="1.0" encoding="utf-8"?>
<!-- Use DrawerLayout as root container for activity -->
<android.support.v4.widget.DrawerLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/drawer_layout"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true">

    <!-- Layout to contain contents of main body of screen (drawer will slide over this) -->
    <FrameLayout
        android:id="@+id/content_frame"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

    <!-- Container for contents of drawer - use NavigationView to make configuration easier -->
    <android.support.design.widget.NavigationView
        android:id="@+id/nav_view"
        android:layout_width="wrap_content"
        android:layout_height="match_parent"
        android:layout_gravity="start"
        android:fitsSystemWindows="true"
        app:menu="@menu/drawer_view"
        app:headerLayout="@layout/nav_header" />

</android.support.v4.widget.DrawerLayout>
```

menu:

```xml
<?xml version="1.0" encoding="utf-8"?>
<menu xmlns:android="http://schemas.android.com/apk/res/android" >
    <group android:checkableBehavior="single">
        <item
            android:id="@+id/nav_camera"
            android:icon="@drawable/ic_menu_camera"
            android:title="@string/import" />
        <item
            android:id="@+id/nav_gallery"
            android:icon="@drawable/ic_menu_gallery"
            android:title="@string/gallery" />
        <item
            android:id="@+id/nav_slideshow"
            android:icon="@drawable/ic_menu_slideshow"
            android:title="@string/slideshow" />
        <item
            android:id="@+id/nav_manage"
            android:icon="@drawable/ic_menu_manage"
            android:title="@string/tools" />
    </group>
</menu>
```

HeaderLayout：

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="192dp"
    android:background="?attr/colorPrimaryDark"
    android:padding="16dp"
    android:theme="@style/ThemeOverlay.AppCompat.Dark"
    android:orientation="vertical"
    android:gravity="bottom">

    <TextView
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="My header title"
        android:textAppearance="@style/TextAppearance.AppCompat.Body1"/>

</LinearLayout>
```

Activity:

```java
public class MainActivity extends AppCompatActivity {

    private DrawerLayout drawerLayout;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        drawerLayout = findViewById(R.id.drawer_layout);

        NavigationView navigationView = findViewById(R.id.nav_view);
        navigationView.setNavigationItemSelectedListener(
                new NavigationView.OnNavigationItemSelectedListener() {
                    @Override
                    public boolean onNavigationItemSelected(MenuItem menuItem) {
                        // set item as selected to persist highlight
                        menuItem.setChecked(true);
                        // close drawer when item is tapped
                        drawerLayout.closeDrawers();

                        // Add code here to update the UI based on the item selected
                        // For example, swap UI fragments here

                        return true;
                    }
                });
    }
}
```

## DataManagement:

 
