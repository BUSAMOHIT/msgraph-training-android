<!-- markdownlint-disable MD002 MD041 -->

In this exercise you will extend the application from the previous exercise to support authentication with Azure AD. This is required to obtain the necessary OAuth access token to call the Microsoft Graph. In this step you will integrate the [Microsoft Authentication Library (MSAL) for Android](https://github.com/AzureAD/microsoft-authentication-library-for-android) into the application.

Right-click the **app/res/values** folder and choose **New**, then **Values resource file**. Name the file `oauth_strings` and choose **OK**. Add the following values to the `resources` element.

```xml
<string name="oauth_app_id">YOUR_APP_ID_HERE</string>
<string name="oauth_redirect_uri">msalYOUR_APP_ID_HERE</string>
<string-array name="oauth_scopes">
    <item>User.Read</item>
    <item>Calendars.Read</item>
</string-array>
```

> [!IMPORTANT]
> If you're using source control such as git, now would be a good time to exclude the `oauth_strings.xml` file from source control to avoid inadvertently leaking your app ID.

## Implement sign-in

Expand the **app/manifests** folder and open **AndroidManifest.xml**. Add the following elements above the `application` element.

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
```

These permissions are required in order for the MSAL library to authenticate the user.

Now add the following element inside the `application` element.

```xml
<activity android:name="com.microsoft.identity.client.BrowserTabActivity">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />

        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />

        <data
            android:host="auth"
            android:scheme="@string/oauth_redirect_uri" />
    </intent-filter>
</activity>
```

This allows MSAL to use a browser to authenticate the user, and registers your redirect URI as being handled by the app.

### Implement an authentication helper

Right-click the **app/java/com.example.graphtutorial** folder and choose **New**, then **Java Class**. Name the class `AuthenticationHelper` and choose **OK**. Open the new file and replace its contents with the following.

```java
package com.example.graphtutorial;

import android.app.Activity;
import android.content.Context;
import android.content.Intent;

import com.microsoft.identity.client.AuthenticationCallback;
import com.microsoft.identity.client.IAccount;
import com.microsoft.identity.client.PublicClientApplication;

// Singleton class - the app only needs a single instance
// of PublicClientApplication
public class AuthenticationHelper {
    private static AuthenticationHelper INSTANCE = null;
    private PublicClientApplication mPCA = null;
    private String[] mScopes;

    private AuthenticationHelper(Context ctx) {
        String appId = ctx.getResources().getString(R.string.oauth_app_id);
        mScopes = ctx.getResources().getStringArray(R.array.oauth_scopes);
        mPCA = new PublicClientApplication(ctx, appId);
    }

    public static synchronized AuthenticationHelper getInstance(Context ctx) {
        if (INSTANCE == null) {
            INSTANCE = new AuthenticationHelper(ctx);
        }

        return INSTANCE;
    }

    public boolean hasAccount() {
        return !mPCA.getAccounts().isEmpty();
    }

    public void handleRedirect(int requestCode, int resultCode, Intent data) {
        mPCA.handleInteractiveRequestRedirect(requestCode, resultCode, data);
    }

    public void acquireTokenInteractively(Activity activity, AuthenticationCallback callback) {
        mPCA.acquireToken(activity, mScopes, callback);
    }

    public void acquireTokenSilently(AuthenticationCallback callback) {
        mPCA.acquireTokenSilentAsync(mScopes, mPCA.getAccounts().get(0), callback);
    }

    public void signOut() {
        for (IAccount account : mPCA.getAccounts()) {
            mPCA.removeAccount(account);
        }
    }
}
```

Now update `MainActivity` to use this new class. Open **MainActivity** and add the following `import` statements.

```java
import android.content.Intent;
import android.support.annotation.Nullable;
import android.util.Log;

import com.microsoft.identity.client.AuthenticationCallback;
import com.microsoft.identity.client.AuthenticationResult;
import com.microsoft.identity.client.exception.MsalClientException;
import com.microsoft.identity.client.exception.MsalException;
import com.microsoft.identity.client.exception.MsalServiceException;
import com.microsoft.identity.client.exception.MsalUiRequiredException;
```

Next, add the following member property to the `MainActivity` class.

```java
private AuthenticationHelper mAuthHelper = null;
```

Update `onCreate` to set `mAuthHelper`. Add the following to the end of the `onCreate` function.

```java
// Get the authentication helper
mAuthHelper = AuthenticationHelper.getInstance(getApplicationContext());
```

Add an override for `onActivityResult` to handle authentication responses.

```java
@Override
protected void onActivityResult(int requestCode, int resultCode, @Nullable Intent data) {
    super.onActivityResult(requestCode, resultCode, data);

    mAuthHelper.handleRedirect(requestCode, resultCode, data);
}
```

Now, add the following functions to the `MainActivity` class.

```java
// Silently sign in - used if there is already a
// user account in the MSAL cache
private void doSilentSignIn() {
    mAuthHelper.acquireTokenSilently(getAuthCallback());
}

// Prompt the user to sign in
private void doInteractiveSignIn() {
    mAuthHelper.acquireTokenInteractively(this, getAuthCallback());
}

// Handles the authentication result
public AuthenticationCallback getAuthCallback() {
    return new AuthenticationCallback() {

        @Override
        public void onSuccess(AuthenticationResult authenticationResult) {
            // Log the token for debug purposes
            String accessToken = authenticationResult.getAccessToken();
            Log.d("AUTH", String.format("Access token: %s", accessToken));
            hideProgressBar();

            setSignedInState(true);
            openHomeFragment(mUserName);
        }

        @Override
        public void onError(MsalException exception) {
            // Check the type of exception and handle appropriately
            if (exception instanceof MsalUiRequiredException) {
                Log.d("AUTH", "Interactive login required");
                doInteractiveSignIn();

            } else if (exception instanceof MsalClientException) {
                // Exception inside MSAL, more info inside MsalError.java
                Log.e("AUTH", "Client error authenticating", exception);
            } else if (exception instanceof MsalServiceException) {
                // Exception when communicating with the auth server, likely config issue
                Log.e("AUTH", "Service error authenticating", exception);
            }
            hideProgressBar();
        }

        @Override
        public void onCancel() {
            // User canceled the authentication
            Log.d("AUTH", "Authentication canceled");
            hideProgressBar();
        }
    };
}
```

Finally, replace the existing `signIn` and `signOut` functions with the following.

```java
private void signIn() {
    showProgressBar();
    if (mAuthHelper.hasAccount()) {
        doSilentSignIn();
    } else {
        doInteractiveSignIn();
    }
}

private void signOut() {
    mAuthHelper.signOut();

    setSignedInState(false);
    openHomeFragment(mUserName);
}
```

Notice that the `signIn` method first checks if there is a user account already in the MSAL cache. If there is, it attempts to refresh its tokens silently, avoiding having to prompt the user every time they launch the app.

Save your changes and run the app. When you tap the **Sign in** menu item, a browser opens to the Azure AD login page. Sign in with your account. Once the app resumes, you should see an access token printed in the debug log in Android Studio.

![A screenshot of the Logcat window in Android Studio](./images/debugger-access-token.png)

## Get user details

Start by creating a helper class to hold all of the calls to Microsoft Graph. Right-click the **app/java/com.example.graphtutorial** folder and choose **New**, then **Java Class**. Name the class `GraphHelper` and choose **OK**. Open the new file and replace its contents with the following.

```java
package com.example.graphtutorial;

import android.content.Context;

import com.microsoft.graph.authentication.IAuthenticationProvider;
import com.microsoft.graph.concurrency.ICallback;
import com.microsoft.graph.http.IHttpRequest;
import com.microsoft.graph.models.extensions.IGraphServiceClient;
import com.microsoft.graph.models.extensions.User;
import com.microsoft.graph.requests.extensions.GraphServiceClient;

// Singleton class - the app only needs a single instance
// of the Graph client
public class GraphHelper implements IAuthenticationProvider {
    private static GraphHelper INSTANCE = null;
    private IGraphServiceClient mClient = null;
    private Context mContext = null;
    private String mAccessToken = null;

    private GraphHelper(Context ctx) {
        mContext = ctx;
        mClient = GraphServiceClient.builder()
                .authenticationProvider(this).buildClient();
    }

    public static synchronized GraphHelper getInstance(Context ctx) {
        if (INSTANCE == null) {
            INSTANCE = new GraphHelper(ctx);
        }

        return INSTANCE;
    }

    // Part of the Graph IAuthenticationProvider interface
    // This method is called before sending the HTTP request
    @Override
    public void authenticateRequest(IHttpRequest request) {
        // Add the access token in the Authorization header
        request.addHeader("Authorization", "Bearer " + mAccessToken);
    }

    public void getUser(String accessToken, ICallback<User> callback) {
        mAccessToken = accessToken;

        // GET /me (logged in user)
        mClient.me().buildRequest().get(callback);
    }
}
```

Note what this code does.

- It implements the `IAuthenticationProvider` interface to insert the access token in the `Authorization` header on outgoing HTTP requests.
- It exposes a `getUser` function to get the logged-in user's information from the `/me` Graph endpoint.

Now update the `MainActivity` class to use this new class to get the logged-in user. Add the following `import` statements to the top of the **MainActivity** file.

```java
import com.microsoft.graph.concurrency.ICallback;
import com.microsoft.graph.core.ClientException;
import com.microsoft.graph.models.extensions.IGraphServiceClient;
import com.microsoft.graph.models.extensions.User;
```

Next, add the following function to the `MainActivity` class to generate an `ICallback` for the Graph call.

```java
private ICallback<User> getUserCallback() {
    return new ICallback<User>() {
        @Override
        public void success(User user) {
            Log.d("AUTH", "User: " + user.displayName);

            mUserName = user.displayName;
            mUserEmail = user.mail == null ? user.userPrincipalName : user.mail;

            runOnUiThread(new Runnable() {
                @Override
                public void run() {
                    hideProgressBar();

                    setSignedInState(true);
                    openHomeFragment(mUserName);
                }
            });

        }

        @Override
        public void failure(ClientException ex) {
            Log.e("AUTH", "Error getting /me", ex);
            mUserName = "ERROR";
            mUserEmail = "ERROR";

            runOnUiThread(new Runnable() {
                @Override
                public void run() {
                    hideProgressBar();

                    setSignedInState(true);
                    openHomeFragment(mUserName);
                }
            });
        }
    };
}
```

Remove the following lines that set the user name and email:

```java
// For testing
mUserName = "Megan Bowen";
mUserEmail = "meganb@contoso.com";
```

Finally, replace the `onSuccess` override in the `AuthenticationCallback` with the following.

```java
@Override
public void onSuccess(AuthenticationResult authenticationResult) {
    // Log the token for debug purposes
    String accessToken = authenticationResult.getAccessToken();
    Log.d("AUTH", String.format("Access token: %s", accessToken));

    // Get Graph client and get user
    GraphHelper graphHelper = GraphHelper.getInstance(getApplicationContext());
    graphHelper.getUser(accessToken, getUserCallback());
}
```

If you save your changes and run the app now, after sign-in the UI is updated with the user's display name and email address.
