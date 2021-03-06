안드로이드 앱에서 웹뷰를 다루던 중, 웹뷰 화면의 EditText를 클릭해서 글자를 적으면 아래와 같은 에러가 발생하였다.

```log
W/System.err: java.lang.NullPointerException: Attempt to invoke virtual method 'boolean android.graphics.drawable.Drawable.getPadding(android.graphics.Rect)' on a null object reference
W/System.err:     at org.chromium.ui.DropdownPopupWindow.<init>(DropdownPopupWindow.java:81)
W/System.err:     at org.chromium.ui.autofill.AutofillPopup.<init>(AutofillPopup.java:48)
W/System.err:     at org.chromium.android_webview.AwAutofillClient.showAutofillPopup(AwAutofillClient.java:50)
W/System.err:     at org.chromium.base.SystemMessageHandler.nativeDoRunLoopOnce(Native Method)
W/System.err:     at org.chromium.base.SystemMessageHandler.handleMessage(SystemMessageHandler.java:39)
W/System.err:     at android.os.Handler.dispatchMessage(Handler.java:102)
W/System.err:     at android.os.Looper.loop(Looper.java:154)
W/System.err:     at android.app.ActivityThread.main(ActivityThread.java:6141)
W/System.err:     at java.lang.reflect.Method.invoke(Native Method)
W/System.err:     at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:912)
W/System.err:     at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:802)
A/chromium: [FATAL:jni_android.cc(236)] Please include Java exception stack in crash report
    --------- beginning of crash
A/google-breakpad: -----BEGIN BREAKPAD MICRODUMP-----
A/google-breakpad: V WebView:52.0.2743.100
A/google-breakpad: O A arm 04 armv7l Android/rk3288/rk3288:7.1.2/NHG47K/starwa07231554:userdebug/test-keys
A/google-breakpad: G OpenGL ES 3.2 v1.r18p0-01rel0.af96b5aad9c0ccc8dee4b7c083ab1938|ARM|Mali-T760
A/google-breakpad: H 12C00000 FFFF1000 006C 40FF8000 9C9CA000 0C:0F 0D:0D 0E:08 0F:07 10:06 11:02 12:03 13:02 14:03 15:03 16:0D 17:0D 18:09 19:06 1A:01 1B:01 1C:02 1E:01
A/google-breakpad: S 0 BE9AB3A0 BE9AB000 00003000
A/google-breakpad: S BE9AB000
```

코드상의 문제가 아닌 웹뷰 자체에서 뱉는 에러라 원인파악이 어려웠다. 먼저 **[FATAL:jni_android.cc(236)]** 을 구글링 해서 찾아본 해결법은 `onDestroyView()` 될 때, `WebView`를 `destroy()` 하라는 것이었다.
</br>


### 해결법 1. WebView destroy()
---
```java
@Override
public void onDestroyView() {
    super.onDestroyView();
    if (myWbView != null) {
        myWbView.destroy();
        webView=null; // remove webView, prevent chromium to crash
    }
}
```
검색했을 때 가장 많은 답변이 `onDestroy()`를 하라는 것이었다.
[https://stackoverflow.com/questions/31416568/could-someone-help-me-with-this-crash-report](https://stackoverflow.com/questions/31416568/could-someone-help-me-with-this-crash-report)

- `destroy()`를 하는 이유? 
Webview 를 xml layout으로 잡을 경우 메모리 누수가 발생할 수 있다. 그렇게 때문에 onDestroy 에서 webview 를 명시적으로 해제 시켜야 함.(혹은 xml이 아닌 코드상으로 WebView객체를 직접 만들어 써야함)

그런데 나는 `destroy()`를 해도 계속 위 에러가 발생하였고, **결국 찾아낸 해결법은 `webView.settings.saveFormData = false` 속성을 주는 것**이었다.
</br>

### 해결법2. saveFormData 에 false값 주기
---
```
W/System.err: java.lang.NullPointerException: Attempt to invoke virtual method 'boolean android.graphics.drawable.Drawable.getPadding(android.graphics.Rect)' on a null object reference
W/System.err:     at org.chromium.ui.DropdownPopupWindow.<init>(DropdownPopupWindow.java:81)
```

`A/chromium: [FATAL:jni_android.cc(236)]` FATAL 에러가 나기 전에, Rect를 그리는 그래픽 객체에 **NullPointerException**이 뜬다. *DropdownPopupWindow* 를 생성하지 못하는데, 내 추측으로는 해당 객체가 EditText를 눌렀을때 호출되는데 버전문제로 인해서 생성하지 못하여 크래쉬가 난 게 아닌가 싶다.(참고로 안드로이드7 버전이었다.)

`A/chromium: [FATAL:jni_android.cc(236)]` 에러가 100% DropdownPopupWindow의 init이 안되어 나는 에러인지는 모르겠는데, 만약 나처럼 특정 객체에 NullPointerException이 발생하여 이 에러가 뜬다면, 해당 오브젝트를 호출하는 웹뷰 속성값을 해제시켜주는 메소드가 있는지 찾아보길 바란다.

나는 `webView.settings.saveFormData = false` 속성을 웹뷰에 적용했더니 문제가 해결되었다. 참고로 코틀린 기준이니 자바는 조금 다를 수 있다.

</br>


