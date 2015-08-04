# Local Notifications on iOS

**Note: This blog post is currently a work in progress.**

Remote notifications, also known as *push notifications*, are a mechanism for informing the user of an app about a content update. They are especially useful in creating engagement with an app and driving retention when used correctly. In order to implement push notifications, some server-side architecture is required including a database to store the tokens needed to send a notification to each registered device. 

There exists another type of notification mechanism on iOS which aren't as well known called local notifications. Local notifications do not require server-side architecture - they are scheduled by an app while it is running rather than being received from Apple's Push Notifcation Service (APNS). Local notifications are indistinguishable from remote notifications to the user - both are displayed in the Notification Centre on an iOS device. The two types of notification mechanism together are referred to as [user notifications](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/Chapters/IPhoneOSClientImp.html).

This blog post will provide a short tutorial on how to implement local notifications in an app. Sample code is available to accompany this post on [GitHub](https://github.com/rwbutler/local-notifications-example).

## Registering for user notifications

Before we may send a local notification, we need to ask the user's permission to do so. This step was not required prior to iOS 8. Previously, the user's permission was only required when to send remote notifications. To send the user a remote notification on iOS 7 and below, we would invoke the method `registerForRemoteNotificationTypes:` of [`UIApplication`](https://developer.apple.com/library/ios/documentation/UIKit/Reference/UIApplication_Class/index.html#//apple_ref/occ/instm/UIApplication/registerUserNotificationSettings:) however now that the user's permission is required to send both types of notifications, the way that we register for both is by invoking `registerUserNotificationSettings:`.

When invoking this method, you will need to specify which permissions you will require. Options include:

- UIUserNotificationTypeAlert - *where the app wishes to post an alert*
- UIUserNotificationTypeBadge - *where the app wishes to badge its icon*
- UIUserNotificationTypeSound - *where the app wishes to play a sound*

You may request only one of these permissions, two, or all of the above as follows:


        [[UIApplication sharedApplication] registerUserNotificationSettings:
         [UIUserNotificationSettings settingsForTypes:
          UIUserNotificationTypeAlert|
          UIUserNotificationTypeBadge|
          UIUserNotificationTypeSound categories:nil]
         ];
        

If you need to support versions of iOS prior to iOS 8 then you should check to see whether `registerUserNotificationSettings:` exists. This can be achieved using the line:

    [UIApplication instancesRespondToSelector:@selector(registerUserNotificationSettings:)]

Our request for permissions to send local notifications has been implemented in [`PermissionsRequestViewController.m`](https://github.com/rwbutler/local-notifications-example/blob/master/LocalNotificationsExample/PermissionsRequestViewController.m).

The first time your app invokes this method, iOS will display the familiar *[Your app name] Would Like to Send You Notifications* dialog. 

## Receiving notification of user notification settings registration

The next step is to implement the method `didRegisterUserNotificationSettings:` of the `UIApplicationDelegate` protocol. With this method definition implemented in your application delegate, regardless of whether the user selects *OK* or *Don't Allow* in response to the permissions dialog presented by iOS, the method will be invoked supplying your application with a [`UIUserNotificationSettings`](https://developer.apple.com/library/prerelease/ios/documentation/UIKit/Reference/UIUserNotificationSettings_class/index.html#//apple_ref/c/tdef/UIUserNotificationType) object indicating whether the user has granted your request to send local notifications or not. 

Setting a breakpoint within the method and then invoking the command `po notificationSettings` will print something akin to the following should the user grant your request for permissions:

    <UIUserNotificationSettings: 0x7fdf0bf60b10; types: (UIUserNotificationTypeAlert UIUserNotificationTypeBadge UIUserNotificationTypeSound);>
    
If the request is denied then the `types` property will contain *none* indicating that none of the permissions you have requested have been granted as follows:

    <UIUserNotificationSettings: 0x7f84d34d2710; types: (none);>

## Asking the user for permission

A significant challenge for iOS developers is the fact that having requested permissions, should the user deny the request, there is no easy way to request permissions again in the future. The only way for the user to grant the app permissions at a later date is through the *Settings* app which third-party applications cannot link to therefore users must find their way to the relevant settings view.

A common practice these days is to display a dialog or view prior to registering user notfication settings explaining why the app is requesting permissions. This approach typically leads to the a much higher number of users granting the app permissions to send notifications. Brenden Mulligan's TechCrunch article the [The Right Way To Ask Users For iOS Permissions](http://techcrunch.com/2014/04/04/the-right-way-to-ask-users-for-ios-permissions/) provides further insight into this approach.

This strategy has been implemented in our sample app. The user is initially presented with the initial view controller named `ViewController.m`. The view contains a button labelled *Schedule a notification in five seconds*. On button touch, the app presents another view controller named `PermissionsRequestViewController.m`. 

This view explains that the user will be presented with a prompt asking whether they will grant permission for the app to send notifications. There are two buttons on this view - one labelled 'Yes', the other labelled 'No, not right now'. Should the user select yes then the app will request permissions to send notifications, otherwise the app does not request permissions and dismisses the current view controller returning the user to the original view. In this way, we reduce the number of users who decline the iOS permissions prompt.

## Remembering whether we have asked for permission

Our initial view controller lists the notifications permissions the user has granted the app. If the app hasn't yet requested permission from the user to send notifications then the result is exactly the same as though we have been denied permission - the `types` property of the `UIUserNotificationSettings` parameter in `didRegisterUserNotificationSettings:` will be set to *none*.

Therefore, we need to record whether or not we have asked for permissions in order to determine whether we have yet to request permissions or have requested permissions and the request has been denied.

The simplest means of achieving this is probably to use [`NSUserDefaults`](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Classes/NSUserDefaults_Class/) to store whether or not we have asked for permissions.

Therefore in `PermissionsRequestViewController.m` after invoking `registerUserNotificationSettings:` we have the following lines to record that we have now requested permissions to send notifications:

            // Set user default indicating that we have requested permissions to send local notifications
        
        [[NSUserDefaults standardUserDefaults] setBool: YES forKey: @"RequestedPermissions"];
        
In combination with a call to `[[UIApplication sharedApplication] currentUserNotificationSettings]` we can now determine whether permissions have been denied or whether we have yet to request access:

    // Check whether we have any permissions
    
    if(settings.types == UIUserNotificationTypeNone)
    {
        // We do not have permissions - check whether we have requested permission
        
        if([[NSUserDefaults standardUserDefaults] objectForKey: @"RequestedPermissions"] == nil){
            
            // We have not yet requested permissions
            
            [_permissionsView setText:@"Permissions not requested"];
            
        } else {
            
            // We have requested permissions but request has been denied
            
            [_permissionsView setText:@"Permissions requested - denied"];
            
        }
    }

## Sending a local notification

Having checked that we have permissions to send notifications and assuming that our request has been granted, we are now ready to send the local notification itself.

In our sample app we schedule a notification to be triggered five seconds from button tap as follows:

    // Create a local notification
    
    UILocalNotification* localNotification = [[UILocalNotification alloc] init];
    
    // Schedule for five seconds from now
    
    localNotification.fireDate = [NSDate dateWithTimeIntervalSinceNow:5];
    localNotification.alertTitle = @"Local Notifications Example";
    localNotification.alertBody = @"Here's a test notification!";
    localNotification.timeZone = [NSTimeZone defaultTimeZone];
    localNotification.applicationIconBadgeNumber = 1;
    localNotification.soundName = UILocalNotificationDefaultSoundName;
    [[UIApplication sharedApplication] scheduleLocalNotification:localNotification];

If you choose to badge the application, play a sound or display an alert as in the above then ensure that you have obtained the relevant permissions when invoking `registerUserNotificationSettings:`.

Since our sample app will likely still be in the foreground when we receive our local notification, we also implement `didReceiveLocalNotification:` of `UIApplicationDelegate` and display an alert indicating that we have received a notification, directing the user to check the notification centre.

     // Create an alert indicating that we have received a local notification
    
    UIAlertView* alertView = [[UIAlertView alloc] initWithTitle:@"Notification Received"
                                                        message:@"Pull down to open notification center and view the notification!" delegate:nil
                                              cancelButtonTitle:@"OK"
                                              otherButtonTitles:nil];
    // Show the alert
    
    [alertView show];
    
By now you should have a fully working example app sending local notifications. If in doubt, take a look at and try building the sample code on [GitHub](https://github.com/rwbutler/local-notifications-example). 

