# 원문
    
  ## All About PendingIntents

  PendingIntents are an important part of the Android framework, but most of the available developer resources focus on their implementation details — a “reference to a token maintained by the system” — rather than their usage.

  Since Android 12 includes important changes to pending intents, including a change that requires explicitly deciding when a PendingIntent is mutable or immutable, I thought it would be helpful to talk more what pending intents do, how the system uses them, and why you might occasionally want a mutable PendingIntent.

  ### What is a PendingIntent?
  A PendingIntent object wraps the functionality of an Intent object while allowing your app to specify something that another app should do, on your app’s behalf, in response to a future action. For example, the wrapped intent might be invoked when an alarm goes off, or when the user taps on a notification.

  A key aspect of pending intents is that another app invokes the intent on your app’s behalf. That is, the other app uses your app’s identity when invoking the intent.

  In order for the PendingIntent to have the same behavior as if it were a normal Intent, the system triggers the PendingIntent with the same identity as it was created with. In most situations, such as the alarm and notifications, this is the identity of the app itself.

  Let’s take a look at the different ways our apps can work with PendingIntents and why we might want to use them in these ways.

  ### Common case
  The most common, and most basic, way to use a PendingIntent is as the action associated with a notification:

  ```kotlin
  val intent = Intent(applicationContext, MainActivity::class.java).apply {
      action = NOTIFICATION_ACTION
      data = deepLink
  }
  val pendingIntent = PendingIntent.getActivity(
      applicationContext,
      NOTIFICATION_REQUEST_CODE,
      intent,
      PendingIntent.FLAG_IMMUTABLE
  )
  val notification = NotificationCompat.Builder(
          applicationContext,
          NOTIFICATION_CHANNEL
      ).apply {
          // ...
          setContentIntent(pendingIntent)
          // ...
      }.build()
  notificationManager.notify(
      NOTIFICATION_TAG,
      NOTIFICATION_ID,
      notification
  )
  ```

  We can see that we’re constructing a standard type of Intent that will open our app, and then simply wrapping that in a PendingIntent before adding it to our notification.

  In this case, since we have an exact action we know we want to perform, we construct a PendingIntent that cannot be modified by the app we pass it to by utilizing a flag called FLAG_IMMUTABLE.

  After we call NotificationManagerCompat.notify() we’re done. The system will display the notification, and, when the user clicks on it, call PendingIntent.send() on our PendingIntent, starting our app.

  ### Updating an immutable PendingIntent
  You might think that if an app needs to update a PendingIntent that it needs to be mutable, but that’s not always the case! The app that creates a PendingIntent can always update it by passing the flag FLAG_UPDATE_CURRENT:

  ```kotlin
  val updatedIntent = Intent(applicationContext, MainActivity::class.java).apply {
     action = NOTIFICATION_ACTION
     data = differentDeepLink
  }
  // Because we're passing `FLAG_UPDATE_CURRENT`, this updates
  // the existing PendingIntent with the changes we made above.
  val updatedPendingIntent = PendingIntent.getActivity(
     applicationContext,
     NOTIFICATION_REQUEST_CODE,
     updatedIntent,
     PendingIntent.FLAG_IMMUTABLE or PendingIntent.FLAG_UPDATE_CURRENT
  )
  // The PendingIntent has been updated.
  ```

  We’ll talk about why one may want to make a PendingIntent mutable in a bit.

  ### Inter-app APIs
  The common case isn’t only useful for interacting with the system. While it is most common to use startActivityForResult() and onActivityResult() to receive a callback after performing an action, it’s not the only way.

  Imagine an online ordering app that provides an API to allow apps to integrate with it. It might accept a PendingIntent as an extra of its own Intent that’s used to start the process of ordering food. The order app only starts the PendingIntent once the order has been delivered.

  In this case the ordering app uses a PendingIntent rather than sending an activity result because it could take a significant amount of time for the order to be delivered, and it doesn’t make sense to force the user to wait while this is happening.

  We want to create an immutable PendingIntent because we don’t want the online ordering app to change anything about our Intent. We just want them to send it, exactly as it is, when the order’s arrived.

  ### Mutable PendingIntents
  But what if we were the developers for the ordering app, and we wanted to add a feature that would allow the user to type a message that would be sent back to the app that called it? Perhaps to allow the calling app to show something like, “It’s PIZZA TIME!”

  The answer to this is to use a mutable PendingIntent.

  Since a PendingIntent is essentially a wrapper around an Intent, one might think that there would be a method called PendingIntent.getIntent() that one could call to get and update the wrapped Intent, but that’s not the case. So how does it work?

  In addition to the send() method on PendingIntent that doesn’t take any parameters, there are a few other versions, including this version, which accepts an Intent:

  ```kotlin
  fun PendingIntent.send(
      context: Context!, 
      code: Int, 
      intent: Intent?
  )
  ```

  This intent parameter doesn’t replace the Intent that’s contained in the PendingIntent, but rather it is used to fill in parameters from the wrapped Intent that weren’t provided when the PendingIntent was created.

  Let’s look at an example.

  ```kotlin
  val orderDeliveredIntent = Intent(applicationContext, OrderDeliveredActivity::class.java).apply {
     action = ACTION_ORDER_DELIVERED
  }
  val mutablePendingIntent = PendingIntent.getActivity(
     applicationContext,
     NOTIFICATION_REQUEST_CODE,
     orderDeliveredIntent,
     PendingIntent.FLAG_MUTABLE
  )
  ```

  This PendingIntent could be handed over to our online order app. After the delivery is completed, the order app could take a customerMessage and send that back as an intent extra like this:

  ```kotlin
  val intentWithExtrasToFill = Intent().apply {
     putExtra(EXTRA_CUSTOMER_MESSAGE, customerMessage)
  }
  mutablePendingIntent.send(
     applicationContext,
     PENDING_INTENT_CODE,
     intentWithExtrasToFill
  )
  ```

  The calling app will then see the extra EXTRA_CUSTOMER_MESSAGE in its Intent and be able to display the message.

  ### Important considerations when declaring pending intent mutability
  ⚠️ When creating a mutable PendingIntent ALWAYS explicitly set the component that will be started in the Intent. This can be done the way we’ve done it above, by explicitly setting the exact class that will receive it, but it can also be done by calling Intent.setComponent().

  Your app might have a use case where it seems easier to call Intent.setPackage(). Be very careful of the possibility to match multiple components if you do this. It’s better to specify a specific component to receive the Intent if at all possible.

  ⚠️ If you attempt to override the values in a PendingIntent that was created with FLAG_IMMUTABLE will fail silently, delivering the original wrapped Intent unmodified.

  Remember that an app can always update its own PendingIntent, even when they are immutable. The only reason to make a PendingIntent mutable is if another app has to be able to update the wrapped Intent in some way.

  ### Details on flags
  We’ve talked a bit about a few of the flags that can be used when creating a PendingIntent, but there are a few others to cover as well.

  FLAG_IMMUTABLE: Indicates the Intent inside the PendingIntent cannot be modified by other apps that pass an Intent to PendingIntent.send(). An app can always use FLAG_UPDATE_CURRENT to modify its own PendingIntents

  Prior to Android 12, a PendingIntent created without this flag was mutable by default.

  ⚠️ On Android versions prior to Android 6 (API 23), PendingIntents are always mutable.

  🆕 FLAG_MUTABLE: Indicates the Intent inside the PendingIntent should allow its contents to be updated by an app by merging values from the intent parameter of PendingIntent.send().

  ⚠️ Always fill in the ComponentName of the wrapped Intent of any PendingIntent that is mutable. Failing to do so can lead to security vulnerabilities!

  This flag was added in Android 12. Prior to Android 12, any PendingIntents created without the FLAG_IMMUTABLE flag were implicitly mutable.

  FLAG_UPDATE_CURRENT: Requests that the system update the existing PendingIntent with the new extra data, rather than storing a new PendingIntent. If the PendingIntent isn’t registered, then this one will be.

  FLAG_ONE_SHOT: Only allows the PendingIntent to be sent once (via PendingIntent.send()). This can be important when passing a PendingIntent to another app if the Intent inside it should only be able to be sent a single time. This may be related to convenience, or to prevent the app from performing some action multiple times.

  🔐 Utilizing FLAG_ONE_SHOT prevents issues such as “replay attacks”.

  FLAG_CANCEL_CURRENT: Cancels an existing PendingIntent, if one exists, before registering this new one. This can be important if a particular PendingIntent was sent to one app, and you’d like to send it to a different app, potentially updating the data. By using FLAG_CANCEL_CURRENT, the first app would no longer be able to call send on it, but the second app would be.

  ### Receiving PendingIntents
  Sometimes the system or other frameworks will provide a PendingIntent as a return from an API call. One example is the method MediaStore.createWriteRequest() that was added in Android 11.

  ```kotlin
  static fun MediaStore.createWriteRequest(
      resolver: ContentResolver, 
      uris: MutableCollection<Uri>
  ): PendingIntent
  ```

  Just as a PendingIntent created by our app is run with our app’s identity, a PendingIntent created by the system is run with the system’s identity. In the case of this API, this allows our app to start an Activity that can grant write permission to a collection of Uris to our app.

  ### Summary
  We’ve talked about how a PendingIntent can be thought of as a wrapper around an Intent that allows the system, or another app, to start an Intent that one app created, as that app, at some time in the future.

  We also talked about how a PendingIntents should usually be immutable and that doing so doesn’t prevent the app from updating its own PendingIntent objects. The way it can do so is by using the FLAG_UPDATE_CURRENT flag in addition to FLAG_IMMUTABLE.

  We’ve also talked about the precautions we should make — ensuring to fill in the ComponentName of the wrapped Intent — if a PendingIntent is necessary to be mutable.

  Finally, we talked about how sometimes the system, or frameworks, may provide PendingIntents to our app so that we can decide how and when to run them.

  Updates to PendingIntent were just one of the features in Android 12 designed to improve app security. Read about all the changes in the preview here.

  Want to do even more? We encourage you to test your apps on the new developer preview of the OS and provide us feedback on your experiences! 
    
# 번역

  ## PendingIntent에 대한 모든것

  PendingIntent는 안드로이드 프레임워크의 중요한 부분이지만, 대부분의 개발 리소스는 구현 세부 사항인 "시스템에서 유지하는 토큰에 대한 참조"에 대해 다루고 있으며, 사용법에 대해서는 잘 다루지 않습니다.

  안드로이드 12에서는 미래 작업에 대한 PendingIntent가 가변(mutability) 또는 불변(immutable)인지 명시적으로 결정해야 하는 변경 사항을 포함하여 PendingIntent에 중요한 변경 사항이 포함되어 있기 때문에, PendingIntent가 하는 일, 시스템에서 어떻게 사용되는지, 가끔 가변 PendingIntent가 필요한 이유에 대해 더 자세히 이야기하고자 합니다.

  ### PendingIntent란?
  PendingIntent 객체는 Intent 객체의 기능을 래핑하면서 앱이 다른 앱이 특정 동작을 수행하도록 요청할 수 있는 기능을 제공합니다. 예를 들어, 감소기가 울릴 때 또는 사용자가 알림을 탭할 때 래핑된 Intent가 호출될 수 있습니다.

  PendingIntent의 핵심적인 측면은 다른 앱이 Intent를 호출할 때 앱의 대신에 앱의 식별 정보를 사용한다는 것입니다.

  PendingIntent가 일반적인 Intent와 동일한 동작을 가져야 한다면, 시스템은 PendingIntent를 생성할 때와 동일한 식별 정보로 PendingIntent를 트리거합니다. 대부분의 경우, 알람 및 알림과 같은 경우에는 앱 자체의 식별 정보입니다.

  PendingIntent를 사용하는 다양한 방법과 그 방식을 사용하려는 이유에 대해 알아보겠습니다.

  ### 일반적인 경우
  가장 일반적이고 기본적인 방법은 PendingIntent를 알림과 관련된 동작으로 사용하는 것입니다.

  ```kotlin
  val intent = Intent(applicationContext, MainActivity::class.java).apply {
      action = NOTIFICATION_ACTION
      data = deepLink
  }
  val pendingIntent = PendingIntent.getActivity(
      applicationContext,
      NOTIFICATION_REQUEST_CODE,
      intent,
      PendingIntent.FLAG_IMMUTABLE
  )
  val notification = NotificationCompat.Builder(
          applicationContext,
          NOTIFICATION_CHANNEL
      ).apply {
          // ...
          setContentIntent(pendingIntent)
          // ...
      }.build()
  notificationManager.notify(
      NOTIFICATION_TAG,
      NOTIFICATION_ID,
      notification
  )
  ```

  우리는 앱을 열 Intent를 만들고, 그것을 PendingIntent로 감싸 알림에 추가하기만 하면 됩니다.

  이 경우, 수행할 특정한 동작이 있으므로, FLAG_IMMUTABLE이라는 플래그를 사용하여 앱이 전달하는 PendingIntent에서 수정할 수 없도록 합니다.

  NotificationManagerCompat.notify()를 호출한 후에는 끝입니다. 시스템은 알림을 표시하고, 사용자가 클릭할 때 PendingIntent.send()를 호출하여 앱을 시작합니다.

  ### immutable PendingIntent 업데이트하기
  만약 앱이 PendingIntent를 업데이트해야 한다면, 가변(mutability) PendingIntent여야 할 것이라고 생각할 수 있지만, 항상 그렇지는 않습니다! PendingIntent를 생성한 앱은 항상 FLAG_UPDATE_CURRENT 플래그를 사용하여 업데이트할 수 있습니다.

  ```kotlin
  val updatedIntent = Intent(applicationContext, MainActivity::class.java).apply {
     action = NOTIFICATION_ACTION
     data = differentDeepLink
  }
  // FLAG_UPDATE_CURRENT를 사용하기 때문에
  // 이로 인해 기존 PendingIntent가 위에서 수정한 내용으로 업데이트됩니다.
  val updatedPendingIntent = PendingIntent.getActivity(
     applicationContext,
     NOTIFICATION_REQUEST_CODE,
     updatedIntent,
     PendingIntent.FLAG_IMMUTABLE or PendingIntent.FLAG_UPDATE_CURRENT
  )
  // PendingIntent가 업데이트되었습니다.
  ```

  ### Inter-app APIs
  일반적으로 startActivityForResult()와 onActivityResult()를 사용하여 동작을 수행한 후에 콜백을 받지만, 이것만이 유일한 방법은 아닙니다.

  온라인 주문 앱을 상상해보세요. 이 앱은 다른 앱들이 이와 통합하기 위해 사용할 수 있는 API를 제공할 수 있습니다. 주문 앱은 음식 주문 프로세스를 시작하는 데 사용되는 자신의 Intent의 extra로 PendingIntent를 수신할 수 있을 것입니다. 주문 앱은 주문이 완료된 후에야 PendingIntent를 시작합니다.

  이 경우, 주문 앱은 startActivityForResult() 대신 PendingIntent를 사용하므로 주문이 완료되는 데에 상당한 시간이 걸릴 수 있으며, 이 시간 동안 사용자가 기다리도록 하는 것은 합리적이지 않습니다.

  우리는 불변(immutable) PendingIntent를 생성하려고 합니다. 외부의 주문 앱이 우리의 Intent에 대해 아무런 변경도 가하지 않기를 원합니다. 주문이 도착한 그대로 보내기만 하면 됩니다.

  ### 가변(mutability) PendingIntent
  그러나 주문 앱의 개발자라면, 호출한 앱으로 돌아갈 때 "피자 시간입니다!"와 같은 메시지를 입력할 수 있는 기능을 추가하고 싶을 수 있습니다. 이 경우 가변(mutability) PendingIntent를 사용합니다.

  PendingIntent는 사실상 Intent를 래핑한 것이므로, PendingIntent.getIntent()라는 메서드를 호출하여 래핑된 Intent를 가져올 수 있다고 생각할 수 있습니다. 그러나 그렇지 않습니다. 그렇다면 어떻게 작동하는 걸까요?

  PendingIntent의 send() 메서드에는 매개변수 없이 호출되는 메서드 외에도, Intent를 매개변수로 사용하는 다른 몇 가지 버전이 있습니다.

  ```kotlin
  fun PendingIntent.send(
      context: Context!, 
      code: Int, 
      intent: Intent?
  )
  ```

  이 intent 매개변수는 PendingIntent에 포함된 Intent를 대체하지 않고, PendingIntent가 생성될 때 제공되지 않은 wrapped Intent의 매개변수를 채우는 데 사용됩니다.

  다음 예를 살펴보겠습니다.

  ```kotlin
  val orderDeliveredIntent = Intent(applicationContext, OrderDeliveredActivity::class.java).apply {
     action = ACTION_ORDER_DELIVERED
  }
  val mutablePendingIntent = PendingIntent.getActivity(
     applicationContext,
     NOTIFICATION_REQUEST_CODE,
     orderDeliveredIntent,
     PendingIntent.FLAG_MUTABLE
  )
  ```

  이 PendingIntent를 온라인 주문 앱에 전달할 수 있습니다. 배송이 완료된 후, 주문 앱은 customerMessage를 가져와 다음과 같이 intent extra로 전송할 수 있습니다.

  ```kotlin
  val intentWithExtrasToFill = Intent().apply {
     putExtra(EXTRA_CUSTOMER_MESSAGE, customerMessage)
  }
  mutablePendingIntent.send(
     applicationContext,
     PENDING_INTENT_CODE,
     intentWithExtrasToFill
  )
  ```

  호출하는 앱은 그 Intent에서 extra인 EXTRA_CUSTOMER_MESSAGE를 볼 수 있고, 해당 메시지를 표시할 수 있게 될 것입니다.

  ### PendingIntent 가변성 선언 시 고려사항
  ⚠️ 가변(mutability) PendingIntent를 생성할 때는 반드시 시작될 컴포넌트(component)를 명시적으로 설정해야 합니다. 위에서처럼 정확한 클래스를 직접 설정하는 방법으로 할 수도 있지만, Intent.setComponent()를 호출하는 방법으로도 설정할 수 있습니다.

  앱에서 Intent.setPackage()를 호출하는 것이 더 간단하다고 생각될 수 있을 것입니다. 그러나 이렇게 하면 여러 컴포넌트가 일치할 가능성이 있으므로 매우 주의해야 합니다. 가능하면 특정 컴포넌트가 Intent를 수신하도록 지정하는 것이 좋습니다.

  ⚠️ FLAG_IMMUTABLE 플래그로 생성된 PendingIntent의 값을 재정의하려고 시도하면, 변경되지 않은 원래 wrapped Intent가 전달되는 silent failure가 발생합니다.

  불변 PendingIntent는 자체 PendingIntent를 업데이트할 수 있습니다. 다른 앱이 무언가를 변경해야 하는 경우에만 PendingIntent를 가변(mutability)으로 만들어야 합니다.

  ### 플래그에 대한 자세한 내용
  팬딩인텐트(PendingIntent)를 생성할 때 사용할 수 있는 몇 가지 플래그에 대해 이야기했지만, 추가로 다룰 플래그가 있습니다.

  FLAG_IMMUTABLE: 팬딩인텐트 내부의 인텐트는 PendingIntent.send()에 인텐트를 전달하는 다른 앱에서 수정할 수 없음을 나타냅니다. 앱은 자체 팬딩인텐트를 수정하기 위해 항상 FLAG_UPDATE_CURRENT를 사용할 수 있습니다.

  Android 12 이전에는 이 플래그 없이 생성된 팬딩인텐트는 기본적으로 가변(mutable)했습니다.

  ⚠️ Android 6(API 23) 이전의 안드로이드 버전에서는 팬딩인텐트가 항상 가변(mutable)했습니다.

  🆕 FLAG_MUTABLE: 팬딩인텐트 내부의 인텐트는 PendingIntent.send()의 인텐트 매개변수의 값과 병합하여 앱에 의해 업데이트될 수 있도록 허용함을 나타냅니다.

  ⚠️ 가변(mutable)인 팬딩인텐트의 경우 항상 래핑된 인텐트의 ComponentName을 채워야 합니다. 이를 놓치면 보안 취약점이 발생할 수 있습니다!

  이 플래그는 Android 12에 추가되었습니다. Android 12 이전에는 FLAG_IMMUTABLE 플래그 없이 생성된 팬딩인텐트는 암시적으로 가변(mutable)했습니다.

  FLAG_UPDATE_CURRENT: 기존의 팬딩인텐트를 새로운 추가 데이터로 업데이트하도록 시스템에 요청합니다. 새로운 팬딩인텐트를 저장하는 대신 기존의 팬딩인텐트를 업데이트합니다. 팬딩인텐트가 등록되어 있지 않은 경우, 이 플래그가 있는 팬딩인텐트가 등록됩니다.

  FLAG_ONE_SHOT: 팬딩인텐트를 한 번만 전송(PendingIntent.send())할 수 있도록 허용합니다. 팬딩인텐트를 다른 앱에 전달할 때, 해당 팬딩인텐트 내부의 인텐트가 한 번만 전송되어야 하는 경우에 중요합니다. 이는 편의성과 함께 앱이 특정 동작을 여러 번 수행하는 것을 방지하는 데 사용될 수 있습니다.

  🔐 FLAG_ONE_SHOT을 사용하면 "재전송 공격(replay attacks)"과 같은 문제를 방지할 수 있습니다.

  FLAG_CANCEL_CURRENT: 존재하는 팬딩인텐트가 있다면 이를 취소한 후 새로운 팬딩인텐트를 등록합니다. 특정 팬딩인텐트가 한 앱에 전송되었고, 이를 다른 앱에 전송하고 데이터를 업데이트하려는 경우에 중요합니다. FLAG_CANCEL_CURRENT를 사용하면 첫 번째 앱은 더 이상 해당 팬딩인텐트에 대해 send를 호출할 수 없지만, 두 번째 앱은 호출할 수 있습니다.

  ### 팬딩인텐트 수신
  시스템이나 다른 프레임워크에서 API 호출의 반환 값으로 팬딩인텐트를 제공하는 경우가 있습니다. Android 11에서 추가된 MediaStore.createWriteRequest() 메서드가 그 예입니다.

    ```kotlin
    static fun MediaStore.createWriteRequest(
        resolver: ContentResolver, 
        uris: MutableCollection<Uri>
    ): PendingIntent
    ```
  앱이 생성한 팬딩인텐트는 앱의 식별자로 실행되는 것처럼, 시스템이 생성한 팬딩인텐트는 시스템의 식별자로 실행됩니다. 이 API의 경우, 앱은 해당 앱에 대해 일괄적으로 여러 URI에 대한 쓰기 권한을 부여할 수 있는 액티비티를 시작할 수 있습니다.

  ### 요약
  팬딩인텐트는 앱이 어떤 시점에서 나중에 앱으로서 시작할 수 있도록 시스템이나 다른 앱이 생성한 인텐트를 래핑하는 것으로 생각할 수 있습니다.

  또한, 팬딩인텐트는 일반적으로 불변(immutable)해야 하며, 이를 통해 앱은 자체 팬딩인텐트 객체를 업데이트하는 것을 방지하지 않습니다. 앱은 FLAG_UPDATE_CURRENT 플래그와 FLAG_IMMUTABLE에 추가하여 자체 팬딩인텐트 객체를 업데이트할 수 있습니다.

  팬딩인텐트가 가변(mutable)해야 하는 경우에는 래핑된 인텐트의 ComponentName을 채우는 등의 주의사항이 있습니다.

  마지막으로, 시스템이나 프레임워크가 팬딩인텐트를 앱에 제공하여 실행할 때와 실행 타이밍을 결정할 수 있도록 하는 경우가 있습니다.

  팬딩인텐트에 대한 업데이트는 Android 12에서 앱 보안을 개선하기 위해 수행된 변경 사항 중 하나입니다. 미리보기에서의 모든 변경 사항에 대해 여기에서 읽어볼 수 있습니다.

  더 많은 작업을 수행하고 싶으신가요? 새로운 OS 개발자 미리보기에서 앱을 테스트하고 경험에 대한 피드백을 제공하는 것을 권장합니다!
