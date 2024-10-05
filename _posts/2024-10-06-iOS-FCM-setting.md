---
title: iOS에서 Push Notification을 위한 환경 세팅하기 (with FCM)
date: 2024-10-06 02:59:00 +0900
categories: [iOS, FCM]
tags: [ios, firebase, push-notification, fcm]     # TAG names should always be lowercase
author: Suyeon
---

# 개요
POCHAK은 푸시 알림 기능이 따로 없이, 앱에 들어가서 알림 탭 확인을 통해 알림을 확인하는 식으로 구현되어있습니다. 포착하는 순간은 같이 있을테니 저희에게 푸시 알림의 중요성은 그렇게 높지 않았죠!

그러나 좋아요나 댓글과 같은 경우.. 푸시 알림이 없으면 확인이 힘들기 때문에, 앱의 편의성을 증대하고자 푸시 알림을 구현하기로 하였습니다.

애플 개발자 계정과 Firebase에 프로젝트가 등록되어있다는 전제 하에 작성하도록 하겠습니다.

# 1. 애플 개발자 계정 설정 변경

## 1-1. Identifiers
Certificates, Identifiers & Profiles > Identifiers 에 들어가서 푸시 알림을 설정하려는 앱을 선택합니다.
![img](/assets/img/2024-10-06-iOS-FCM-setting/img0.png)

Push Notifications를 찾아서 체크해주고, Save! (나중에 헷갈릴까봐 name도 조금 수정했습니다 ㅎㅎ)
![img](/assets/img/2024-10-06-iOS-FCM-setting/img1.png)

이미 있는 identifier 값을 변경하려고 하니 다음과 같은 창이 뜨는데... 그냥 컨펌해줍니다.
![img](/assets/img/2024-10-06-iOS-FCM-setting/img2.png)

## 1-2. Certificates
인증서를... 다시 발급받아야 합니다. Certificates로 이동해서 새로운 인증서를 만들어줍시다.

아래와 같이 Services 섹션에 있는 `Apple Push Notifications service SSL (Sandbox & Production)`을 선택해줍니다. 위는 개발용, 밑에(선택한 것)는 개발&배포용이라는데... 굳이 개발용과 배포용을 나눌 필요는 없을 것 같아 밑에 것으로 선택해주었습니다.
![img](/assets/img/2024-10-06-iOS-FCM-setting/img3.png)

이후 적용하려는 App ID 선택해주고, Continue.

그 다음, CSR을 업로드해주어야 하는데, 맥북에서 키체인 접근 앱을 엽니다. (`cmd` + `space` > 키체인 접근 검색)

`인증서 지원` > `인증 기관에서 인증서 요청` 선택
![img](/assets/img/2024-10-06-iOS-FCM-setting/img4.png)

본인 이메일 입력하고, 요청 항목을 `디스크에 저장됨` 으로 선택해서 저장합니다.
![img](/assets/img/2024-10-06-iOS-FCM-setting/img5.png)

다음과 같이 잘 생성된 것을 확인할 수 있습니다. 다운로드 버튼을 눌러서 저장합니다.
![img](/assets/img/2024-10-06-iOS-FCM-setting/img6.png)


## 1-3. Key 발급받기
Certificates, Identifiers & Profiles > Keys 에 들어가서 새로운 Key 등록을 시작합니다! (키는 한 아이디당 <strong>두 개만</strong> 생성 가능하다고 합니다.)
![img](/assets/img/2024-10-06-iOS-FCM-setting/img7.png)

까먹고 캡쳐를 못했는데, 알아볼 수 있는 이름을 작성(ex. Key for Push Noti) 하고, Apple Push Notification Service (APNs) 에 체크하고 넘어가서 .p8 파일을 다운받습니다.
<span style="color:red">키 파일은 한 번 다운받은 후에는 다시 다운받지 못하므로 잘 보관해두어야 합니다!!!</span>
![img](/assets/img/2024-10-06-iOS-FCM-setting/img8.png)
다음과 같이 키가 두 개 생겼네요.

# 2. Xcode 프로젝트에서 세팅 변경
Xcode에서 프로젝트를 열고, Targets > Signing & Capabilites 에서 새로운 Capability를 추가해줍니다.
![img](/assets/img/2024-10-06-iOS-FCM-setting/img9.png)

여기서, Push Notifications와 Background Mode를 추가해주면 되는데, Background Mode를 추가하려고 하니 다음과 같은 창이.. 떴습니다. 쿨하게 Change All 누릅니다 ㅎ
![img](/assets/img/2024-10-06-iOS-FCM-setting/img10.png)

그러면 Background Mode 와 관련된 상세한 모드?를 선택할 수 있는데 `Remote notifications`를 선택해줍니다.
![img](/assets/img/2024-10-06-iOS-FCM-setting/img11.png)

# 3. Firebase 프로젝트에 APN key 등록

프로젝트 개요 > 설정 버튼 > 프로젝트 설정 클릭
![img](/assets/img/2024-10-06-iOS-FCM-setting/img12.png)

클라우드 메시징 탭을 열어서 밑으로 내려가 `Apple 앱 구성` 섹션에서 APN 인증 키 업로드를 해줍니다.

이때 키 ID와 팀 ID를 입력해야하는데, 키 ID는 위의 1.3에서 새로운 Key를 만들면서 생성된 KEY ID이고, 팀 ID는 애플 개발자 계정 멤버십의 팀 ID입니다. 
![img](/assets/img/2024-10-06-iOS-FCM-setting/img13.png)

또한, 포착은 현재 FirebaseAnalytics와 FirebaseCrashlytics만 추가된 상태이므로, FirebaseMessaging 프레임워크를 새로 추가해주어야 했습니다. 포착은 Cocoapods를 사용하고 있어, Podfile에 FIRMessaging을 새로 추가했습니다.
![img](/assets/img/2024-10-06-iOS-FCM-setting/img14.png)

터미널을 열어 프로젝트 파일로 이동한 후, `pod install --repo-update` 입력해서 podfile의 수정 사항을 반영합니다. 반영되던 중 Failed to save Pods. 어쩌구.. 창이 떠서.. 고민하다가 Read from Disk를 선택해주었습니다.
![img](/assets/img/2024-10-06-iOS-FCM-setting/img15.png)

여기까지 하면 기본적인 환경 설정은 모두 완료되었습니다!

# 4. AppDelegate 작성
푸시 알림이 오는 앱을 사용하는 경우, 처음 앱을 다운받아 실행하면 푸시 알림에 대한 권한 동의창이 열립니다. 이 부분에 대한 구현이 필요합니다.

`AppDelegate`의 `didFinishiLaunchingWithOptions` 메소드에 다음과 같은 코드를 추가해줍니다. `FirebaseApp.configure()`는 이미 있었던 코드고, 여기 아래에 새 코드를 추가해 주었습니다.

```swift
class AppDelegate: UIResponder, UIApplicationDelegate {
    
    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
	// 앱이 시작될 때 Firebase 연동
	FirebaseApp.configure()
    
    /// 아래부터 새로 추가된 코드
	/// 앱 실행 시 사용자에게 알림 허용 권한 받기
	UNUserNotificationCenter.current().delegate = self
        
	let authOptions: UNAuthorizationOptions = [.alert, .badge, .sound] // 필요한 알림 권한 설정(알람창, 앱에 뱃지, 알람소리)
	UNUserNotificationCenter.current().requestAuthorization(
		options: authOptions,
	    completionHandler: { _, _ in }
	)
        
	/// UNUserNotificationCenterDelegate를 구현한 메소드 실행
    application.registerForRemoteNotifications()
        
	/// Firebase Meesaging delegate 설정
  	Messaging.messaging().delegate = self
        
    /// FCM 발급받은 토큰 가져오기
    Messaging.messaging().token { token, error in
         if let error = error {
             print("Error fetching FCM registration token: \(error)")
         } 
         else if let token = token {
             print("FCM registration token: \(token)")
         }
	}
}

extension AppDelegate: UNUserNotificationCenterDelegate {

    /// 백그라운드에서 푸시 알림을 탭했을 때 실행
    func application(_ application: UIApplication,
                     didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data) {
        print("Background, APNS token: \(deviceToken)")
        Messaging.messaging().apnsToken = deviceToken
    }
    
    /// Foreground(앱 켜진 상태) 에서 알림 오는 설정
    func userNotificationCenter(_ center: UNUserNotificationCenter, willPresent notification: UNNotification,withCompletionHandler completionHandler: @escaping (UNNotificationPresentationOptions) -> Void) {
        print("Foreground, 메시지 수신")
        completionHandler([.alert, .badge, .sound])
    }
}

extension AppDelegate: MessagingDelegate {
    
    /// FCM토큰이 변경되었을 때를 감지, 새로운 토큰으로 갱신해서 저장
    func messaging(_ messaging: Messaging, didReceiveRegistrationToken fcmToken: String?) {
      print("Firebase registration token: \(String(describing: fcmToken))")

      let dataDict: [String: String] = ["token": fcmToken ?? ""]
      NotificationCenter.default.post(
        name: Notification.Name("FCMToken"),
        object: nil,
        userInfo: dataDict
      )
      // TODO: If necessary send token to application server.
      // Note: This callback is fired at each app startup and whenever a new token is generated.
    }

}
```
약간의 설명을 적어보면...

1. ```token(completion:)```을 사용하여 직접 토큰을 가져올 수 있습니다. 이 메소드를 통해 토큰을 따로 저장하지 않고도 토큰에 액세스할 수 있다고 하는데, 저장하는게 속 편할 것 같습니다. 더 찾아볼 예정인데.. 아마 Keychain에 저장하지 않을까 싶네요.

2. `UNUserNotificationCenterDelegate`의 `didRegisterForRemoteNotificationsWithDeviceToken`은 백그라운드에서 푸시 알림을 탭했을 때 실행되는 메소드, `willPresent` 메소드는 앱 안에 있는 상태에서 알림이 올 때를 설정합니다. 특히 `willPresent` 메소드는 구현해놓지 않으면 앱 안에 있을 때는 푸시 알림이 오지 않기 때문에, 사용자가 앱 안에서 머무르고 있을 때에도 푸시 알림이 오길 원한다면 작성해야 합니다.

3. `MessagingDelegate`의 `didReceiveRegistrationToken` 메소드는 FCM에서 발급받은 토큰이 변경되었을 때를 감지하고 앱 내의 NotificationCenter에 새로운 토큰을 저장할 수 있도록 합니다.

Firebase는 예시가 잘 나와있어서 [공식 문서](https://firebase.google.com/docs/cloud-messaging/ios/client?hl=ko) 확인하면 다른 예시 코드도 확인 가능합니다!

# 5. 잘 되는지 테스트해보기
Firebase 콘솔에서 직접 테스트해볼 수도 있고, Swifty Pusher라는 것을 사용해서 푸시 알림 테스트를 해볼 수 있습니다. 저는 firebase 콘솔창에서 테스트해보았습니다.

만약 firebase 콘솔창을 통해 테스트해보신다면.. (혹은 프로젝트의 서버가 따로 존재한다면)

<span style="color:red">위 과정까지 마친 다음 프로젝트를 한 번 실행해서 xcode 콘솔창에 찍히는 FCM token 값을 복사해서 어딘가에 저장해야 합니다!!!!</span> 

콘솔 창을 통한 테스트를 위해 필요하고, 서버 팀원들이 기능 구현을 위해 테스트해보기 위해 토큰이 필요하기 때문입니다.

만약 이미 실행해버렸다면 다시 실행해도 새로 찍히지않을 것이므로.. 앱을 삭제했다가 다시 실행해주면 됩니다. 

다시 콘솔로 이동해서 Messaging > 첫 번째 캠페인 만들기 선택!
![img](/assets/img/2024-10-06-iOS-FCM-setting/img16.png)

Firebase 알림 메시지 선택
![img](/assets/img/2024-10-06-iOS-FCM-setting/img17.png)

푸시 알림 내용을 작성해줍니다.
![img](/assets/img/2024-10-06-iOS-FCM-setting/img18.png)

테스트 메시지 전송을 클릭하고, 앞서 복사해둔 FCM 토큰을 넣어서 보내보면... 알림이 잘 옵니다!! 구현하고 보니 알림 소리가 나질 않는데.. 이건 추후 Rich Push Notification 구현하면서 해결해보려 합니다.
![img](/assets/img/2024-10-06-iOS-FCM-setting/img19.jpg)

끝!
