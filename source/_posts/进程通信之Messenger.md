---
title: 进程通信之Messenger
date: 2017-05-04 15:27:34
tags: Android
---
[Messenger](https://developer.android.com/reference/android/os/Messenger.html)是简化版的AIDL，对AIDL进行了一定的封装。
### 服务端

```java
public class MessengerService extends Service {
    /** Command to the service to display a message */
    static final int MSG_SAY_HELLO = 1;

    /**
     * Handler of incoming messages from clients.
     */
    class IncomingHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_SAY_HELLO:
                    Toast.makeText(getApplicationContext(), "hello!", Toast.LENGTH_SHORT).show();
                    break;
                default:
                    super.handleMessage(msg);
            }
        }
    }

    /**
     * Target we publish for clients to send messages to IncomingHandler.
     */
    final Messenger mMessenger = new Messenger(new IncomingHandler());

    /**
     * When binding to the service, we return an interface to our messenger
     * for sending messages to the service.
     */
    @Override
    public IBinder onBind(Intent intent) {
        Toast.makeText(getApplicationContext(), "binding", Toast.LENGTH_SHORT).show();
        return mMessenger.getBinder();
    }
}
```
在*IncomingHandler*中处理收到的消息

### 客户端
客户端要做的使用服务端返回的*IBinder*对象来创建一个*Messenger*对象，使用*Messenger*的*Send()*方法向服务端发送*Message*消息

```java
public class ActivityMessenger extends Activity {
    /** Messenger for communicating with the service. */
    Messenger mService = null;

    /** Flag indicating whether we have called bind on the service. */
    boolean mBound;

    /**
     * Class for interacting with the main interface of the service.
     */
    private ServiceConnection mConnection = new ServiceConnection() {
        public void onServiceConnected(ComponentName className, IBinder service) {
            // This is called when the connection with the service has been
            // established, giving us the object we can use to
            // interact with the service.  We are communicating with the
            // service using a Messenger, so here we get a client-side
            // representation of that from the raw IBinder object.
            mService = new Messenger(service);
            mBound = true;
        }

        public void onServiceDisconnected(ComponentName className) {
            // This is called when the connection with the service has been
            // unexpectedly disconnected -- that is, its process crashed.
            mService = null;
            mBound = false;
        }
    };

    public void sayHello(View v) {
        if (!mBound) return;
        // Create and send a message to the service, using a supported 'what' value
        Message msg = Message.obtain(null, MessengerService.MSG_SAY_HELLO, 0, 0);
        try {
            mService.send(msg);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main);
    }

    @Override
    protected void onStart() {
        super.onStart();
        // Bind to the service
        bindService(new Intent(this, MessengerService.class), mConnection,
            Context.BIND_AUTO_CREATE);
    }

    @Override
    protected void onStop() {
        super.onStop();
        // Unbind from the service
        if (mBound) {
            unbindService(mConnection);
            mBound = false;
        }
    }
}
```
### 服务端回应
要在服务端收到客户端的消息之后进行回应，可以使用*Message*的[reply](https://developer.android.com/reference/android/os/Message.html#replyTo)参数。

客户端中声明一个*Messenger*对象，设置给发送给服务端的*Message*消息对象中。

```java
public void sayHello(View v) {
    if (!mBound) return;
    // Create and send a message to the service, using a supported 'what' value
    Message msg = Message.obtain(null, MessengerService.MSG_SAY_HELLO, 0, 0);
    msg.replyTo = mMessenger;
    try {
        mService.send(msg);
    } catch (RemoteException e) {
        e.printStackTrace();
    }
}
    
Messenger mMessenger = new Messenger(new MyHandler());
class MyHandler extends Handler{
    @Override
    public void handleMessage(Message msg) {
        super.handleMessage(msg);
        Bundle bundle = msg.getData();
        LLog.d("receive reply in activity : " + bundle.getString("data"));
        LLog.d(msg.toString());
    }
}
```
如果服务端有消息返回，在*Handler*中进行处理。

服务端收到客户端消息后，使用*send()*方法再回应消息。

```java
class IncomingHandler extends Handler{

    @Override
    public void handleMessage(Message msg) {
        super.handleMessage(msg);
        Message replyMsg = Message.obtain();
        Bundle bundle = new Bundle();
        bundle.putString("data", "Message from service");
        replyMsg.setData(bundle);
        try {
            msg.replyTo.send(replyMsg);
        } catch (RemoteException e) {
           e.printStackTrace();
        }
    }
}
```

最后来看看**Messenger**的实现
```java
public final class Messenger implements Parcelable {
    private final IMessenger mTarget;

    /**
     * Create a new Messenger pointing to the given Handler.  Any Message
     * objects sent through this Messenger will appear in the Handler as if
     * {@link Handler#sendMessage(Message) Handler.sendMessage(Message)} had
     * been called directly.
     * 
     * @param target The Handler that will receive sent messages.
     */
    public Messenger(Handler target) {
        mTarget = target.getIMessenger();
    }
    
    /**
     * Send a Message to this Messenger's Handler.
     * 
     * @param message The Message to send.  Usually retrieved through
     * {@link Message#obtain() Message.obtain()}.
     * 
     * @throws RemoteException Throws DeadObjectException if the target
     * Handler no longer exists.
     */
    public void send(Message message) throws RemoteException {
        mTarget.send(message);
    }
    
    /**
     * Retrieve the IBinder that this Messenger is using to communicate with
     * its associated Handler.
     * 
     * @return Returns the IBinder backing this Messenger.
     */
    public IBinder getBinder() {
        return mTarget.asBinder();
    }
    
    /**
     * Comparison operator on two Messenger objects, such that true
     * is returned then they both point to the same Handler.
     */
    public boolean equals(Object otherObj) {
        if (otherObj == null) {
            return false;
        }
        try {
            return mTarget.asBinder().equals(((Messenger)otherObj)
                    .mTarget.asBinder());
        } catch (ClassCastException e) {
        }
        return false;
    }

    public int hashCode() {
        return mTarget.asBinder().hashCode();
    }
    
    public int describeContents() {
        return 0;
    }

    public void writeToParcel(Parcel out, int flags) {
        out.writeStrongBinder(mTarget.asBinder());
    }

    public static final Parcelable.Creator<Messenger> CREATOR
            = new Parcelable.Creator<Messenger>() {
        public Messenger createFromParcel(Parcel in) {
            IBinder target = in.readStrongBinder();
            return target != null ? new Messenger(target) : null;
        }

        public Messenger[] newArray(int size) {
            return new Messenger[size];
        }
    };

    /**
     * Convenience function for writing either a Messenger or null pointer to
     * a Parcel.  You must use this with {@link #readMessengerOrNullFromParcel}
     * for later reading it.
     * 
     * @param messenger The Messenger to write, or null.
     * @param out Where to write the Messenger.
     */
    public static void writeMessengerOrNullToParcel(Messenger messenger,
            Parcel out) {
        out.writeStrongBinder(messenger != null ? messenger.mTarget.asBinder()
                : null);
    }
    
    /**
     * Convenience function for reading either a Messenger or null pointer from
     * a Parcel.  You must have previously written the Messenger with
     * {@link #writeMessengerOrNullToParcel}.
     * 
     * @param in The Parcel containing the written Messenger.
     * 
     * @return Returns the Messenger read from the Parcel, or null if null had
     * been written.
     */
    public static Messenger readMessengerOrNullFromParcel(Parcel in) {
        IBinder b = in.readStrongBinder();
        return b != null ? new Messenger(b) : null;
    }
    
    /**
     * Create a Messenger from a raw IBinder, which had previously been
     * retrieved with {@link #getBinder}.
     * 
     * @param target The IBinder this Messenger should communicate with.
     */
    public Messenger(IBinder target) {
        mTarget = IMessenger.Stub.asInterface(target);
    }
}
```
