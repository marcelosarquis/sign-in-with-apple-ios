# Implementing User Authentication with Sign in with Apple

Provide a way for users of your app to set up an account and start using your services.

## Overview

This sample app, Juice, uses the [AuthenticationServices][1] framework to provide users an interface to set up accounts and sign in with their Apple ID. The app presents a form in which the user can create and set up an account for the app, then authenticates the user’s Apple ID with Sign in with Apple, and displays the user’s account data. 

For more information about implementing Sign in with Apple on iOS 12 and earlier, see [Incorporating Sign in with Apple into Other Platforms][2].

## Configure the Sample Code Project

To configure the sample code project, perform the following steps in Xcode:

1. On the Signing & Capabilities pane, [set the bundle ID][3] to a unique identifier (you must change the bundle ID to proceed).
2. [Add your Apple ID account][4] and [assign the target to a team][5] so Xcode can enable the Sign in with Apple capability with your provisioning profile.
3. Choose a run destination from the scheme pop-up menu that you’re signed into with an Apple ID and that uses Two-Factor Authentication.
4. If necessary, click Register Device in the Signing & Capabilities pane to create the provisioning profile.
5. In the toolbar, click Run, or choose Product > Run (⌘R). 
 
 ## Add a Sign in with Apple Button
 
 In the sample app, `LoginViewController` displays a login form and a Sign in with Apple button ([`ASAuthorizationAppleIDButton`][6]) in its view hierarchy. The view controller also adds itself as the button’s target, and passes an action to be invoked when the button receives a touch-up event.
 
``` swift
func setupProviderLoginView() {
    let authorizationButton = ASAuthorizationAppleIDButton()
    authorizationButton.addTarget(self, action: #selector(handleAuthorizationAppleIDButtonPress), for: .touchUpInside)
    self.loginProviderStackView.addArrangedSubview(authorizationButton)
}
```
 
 [View in Source](x-source-tag://add_appleid_button)
 
 - Important: When adding the Sign in with Apple button to your storyboard, you must also set the control's class value to `ASAuthorizationAppleIDButton` in Xcode's Identity Inspector.
 
 ## Request Authorization with Apple ID
 
 When the user taps the Sign in with Apple button, the view controller invokes the `handleAuthorizationAppleIDButtonPress()` function, which starts the authentication flow by performing an authorization request for the users’s full name and email address. The system then checks whether the user is signed in with their Apple ID on the device. If the user is not signed in at the system-level, the app presents an alert directing the user to sign in with their Apple ID in Settings.

``` swift
@objc
func handleAuthorizationAppleIDButtonPress() {
    let appleIDProvider = ASAuthorizationAppleIDProvider()
    let request = appleIDProvider.createRequest()
    request.requestedScopes = [.fullName, .email]
    
    let authorizationController = ASAuthorizationController(authorizationRequests: [request])
    authorizationController.delegate = self
    authorizationController.presentationContextProvider = self
    authorizationController.performRequests()
}
```

[View in Source](x-source-tag://perform_appleid_request)

- Important: The user must enable Two-Factor Authentication to use Sign in with Apple so that access to the account is secure.
 
 The authorization controller calls the [`ASAuthorizationControllerPresentationContextProviding.presentationAnchor(for:)`][7] function to get the window from the app where it presents the Sign in with Apple content to the user in a modal sheet.
 
``` swift
func presentationAnchor(for controller: ASAuthorizationController) -> ASPresentationAnchor {
    return self.view.window!
}
```
 
 [View in Source](x-source-tag://provide_presentation_anchor)
 
 If the user is signed in at the system-level with their Apple ID, the sheet appears describing the Sign in with Apple feature, followed by another sheet allowing the user to edit the information in their account. The user can edit their first and last name, choose another email address as their contact information, and hide their email address from the app. If the user chooses to hide their email address from the app, Apple generates a proxy email address to forward email to the user’s private email address. Lastly, the user enters the password for the Apple ID, then clicks Continue to create the account.
 
 ## Handle User Credentials
 
 If the authentication succeeds, the authorization controller invokes the [`ASAuthorizationControllerDelegate.authorizationController(controller:didCompleteWithAuthorization:)`][8] delegate function, which the app uses to store the user’s data in the keychain.
 
``` swift
func authorizationController(controller: ASAuthorizationController, didCompleteWithAuthorization authorization: ASAuthorization) {
    switch authorization.credential {
    case let appleIDCredential as ASAuthorizationAppleIDCredential:
        
        // Create an account in your system.
        let userIdentifier = appleIDCredential.user
        let fullName = appleIDCredential.fullName
        let email = appleIDCredential.email
        
        // For the purpose of this demo app, store the `userIdentifier` in the keychain.
        self.saveUserInKeychain(userIdentifier)
        
        // For the purpose of this demo app, show the Apple ID credential information in the `ResultViewController`.
        self.showResultViewController(userIdentifier: userIdentifier, fullName: fullName, email: email)
    
    case let passwordCredential as ASPasswordCredential:
    
        // Sign in using an existing iCloud Keychain credential.
        let username = passwordCredential.user
        let password = passwordCredential.password
        
        // For the purpose of this demo app, show the password credential as an alert.
        DispatchQueue.main.async {
            self.showPasswordCredentialAlert(username: username, password: password)
        }
        
    default:
        break
    }
}
```
 
 [View in Source](x-source-tag://did_complete_authorization)
 
 - Note: In your implementation, the `ASAuthorizationControllerDelegate.authorizationController(controller:didCompleteWithAuthorization:)` delegate function should create an account in your system using the data contained in the user identifier.

If the authentication fails, the authorization controller invokes the [`ASAuthorizationControllerDelegate.authorizationController(controller:didCompleteWithError:)`][9] delegate function to handle the error.

``` swift
func authorizationController(controller: ASAuthorizationController, didCompleteWithError error: Error) {
    // Handle error.
}
```

[View in Source](x-source-tag://did_complete_error)

Once the system authenticates the user, the app displays the `ResultViewController` which shows the user information requested from the framework, including the user-provided full name and email address. The view controller also displays a Sign Out button and stores the user data in the keychain. When the user taps the Sign Out button, the app deletes the user information from the view controller and the keychain, and presents the `LoginViewController` to the user.

## Request Existing Credentials

The `LoginViewController.performExistingAccountSetupFlows()` function checks if the user has an existing account by requesting both an Apple ID and an iCloud keychain password. Similar to `handleAuthorizationAppleIDButtonPress()`, the authorization controller sets its presentation content provider and delegate to the `LoginViewController` object.

``` swift
func performExistingAccountSetupFlows() {
    // Prepare requests for both Apple ID and password providers.
    let requests = [ASAuthorizationAppleIDProvider().createRequest(),
                    ASAuthorizationPasswordProvider().createRequest()]
    
    // Create an authorization controller with the given requests.
    let authorizationController = ASAuthorizationController(authorizationRequests: requests)
    authorizationController.delegate = self
    authorizationController.presentationContextProvider = self
    authorizationController.performRequests()
}
```

[View in Source](x-source-tag://perform_appleid_password_request)

The `authorizationController(controller:didCompleteWithAuthorization:)` delegate function checks whether the credential is an Apple ID ([`ASAuthorizationAppleIDCredential`][10]) or a password credential ([`ASPasswordCredential`][11]). If the credential is a password credential, the system displays an alert allowing the user to authenticate with the existing account.

## Check User Credentials at Launch

The sample app only shows the Sign in with Apple user interface when necessary. The app delegate checks the status of the saved user credentials immediately after launch in the `AppDelegate.application(_:didFinishLaunchingWithOptions:)` function.

The [`ASAuthorizationAppleIDProvider.getCredentialState()`][12] function retrieves the state of the user identifier saved in the keychain. If the user granted authorization for the app (for example, the user is signed into the app with their Apple ID on the device), then the app continues executing. If the user revoked authorization for the app, or the user’s credential state not found, the app displays the log in form by invoking the `showLoginViewController()` function.

``` swift
func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    let appleIDProvider = ASAuthorizationAppleIDProvider()
    appleIDProvider.getCredentialState(forUserID: KeychainItem.currentUserIdentifier) { (credentialState, error) in
        switch credentialState {
        case .authorized:
            break // The Apple ID credential is valid.
        case .revoked, .notFound:
            // The Apple ID credential is either revoked or was not found, so show the sign-in UI.
            DispatchQueue.main.async {
                self.window?.rootViewController?.showLoginViewController()
            }
        default:
            break
        }
    }
    return true
}
```

[View in Source](x-source-tag://did_finish_launching)

[1]:https://developer.apple.com/documentation/authenticationservices
[2]:https://developer.apple.com/documentation/signinwithapplejs/incorporating_sign_in_with_apple_into_other_platforms
[3]:https://help.apple.com/xcode/mac/current/#/deve21d0239c
[4]:https://help.apple.com/xcode/mac/current/#/devaf282080a
[5]:https://help.apple.com/xcode/mac/current/#/dev23aab79b4
[6]:https://developer.apple.com/documentation/authenticationservices/asauthorizationappleidbutton
[7]:https://developer.apple.com/documentation/authenticationservices/asauthorizationcontrollerpresentationcontextproviding/3237228-presentationanchor
[8]:https://developer.apple.com/documentation/authenticationservices/asauthorizationcontrollerdelegate/3153050-authorizationcontroller
[9]:https://developer.apple.com/documentation/authenticationservices/asauthorizationcontrollerdelegate/3153051-authorizationcontroller
[10]:https://developer.apple.com/documentation/authenticationservices/asauthorizationappleidcredential
[11]:https://developer.apple.com/documentation/authenticationservices/aspasswordcredential
[12]:https://developer.apple.com/documentation/authenticationservices/asauthorizationappleidprovider/3175423-getcredentialstate
