# laravel-firebase-integration

# Laravel 12 + FCM Integration Complete Guide

## Step 1: Firebase Account Setup

### 1.1 Create Firebase Project
1. Go to [Firebase Console](https://console.firebase.google.com/)
2. Click "Add Project"
3. Enter project name and follow the setup wizard
4. Enable Google Analytics (optional)

### 1.2 Get Server Key & Credentials
1. In Firebase Console, go to **Project Settings** (gear icon)
2. Navigate to **Service Accounts** tab
3. Click **Generate New Private Key**
4. Download the JSON file (keep it secure)
5. Go to **Cloud Messaging** tab
6. Note your **Server Key** (legacy) or use the JSON file for authentication

---

## Step 2: Laravel Project Setup

### 2.1 Install Required Package

```bash
composer require kreait/laravel-firebase
```

### 2.2 Publish Configuration

```bash
php artisan vendor:publish --provider="Kreait\Laravel\Firebase\ServiceProvider" --tag=config
```

### 2.3 Environment Configuration

Add to `.env`:

```env
FIREBASE_CREDENTIALS=path/to/your-firebase-credentials.json
# Or use base64 encoded credentials
FIREBASE_CREDENTIALS_BASE64=your_base64_encoded_json
```

Place your Firebase JSON file in `storage/app/firebase/` directory.

Update `config/firebase.php`:

```php
return [
    'credentials' => [
        'file' => env('FIREBASE_CREDENTIALS'),
    ],
    'database' => [
        'url' => env('FIREBASE_DATABASE_URL'),
    ],
];
```

---

## Step 3: Database Migration

Create migration for storing device tokens:

```bash
php artisan make:migration create_device_tokens_table
```

Migration file:

```php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('device_tokens', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained()->onDelete('cascade');
            $table->string('token')->unique();
            $table->enum('platform', ['android', 'ios', 'web']);
            $table->boolean('is_active')->default(true);
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('device_tokens');
    }
};
```

Run migration:

```bash
php artisan migrate
```

---

## Step 4: Create Models

### DeviceToken Model

```bash
php artisan make:model DeviceToken
```

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class DeviceToken extends Model
{
    protected $fillable = [
        'user_id',
        'token',
        'platform',
        'is_active'
    ];

    protected $casts = [
        'is_active' => 'boolean',
    ];

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }
}
```

Update User Model:

```php
use Illuminate\Database\Eloquent\Relations\HasMany;

class User extends Authenticatable
{
    // ... existing code

    public function deviceTokens(): HasMany
    {
        return $this->hasMany(DeviceToken::class);
    }
}
```

---

## Step 5: Create FCM Service

```bash
php artisan make:class Services/FCMService
```

`app/Services/FCMService.php`:

```php
namespace App\Services;

use Kreait\Firebase\Factory;
use Kreait\Firebase\Messaging\CloudMessage;
use Kreait\Firebase\Messaging\Notification;
use Kreait\Firebase\Messaging\AndroidConfig;
use Kreait\Firebase\Messaging\WebPushConfig;
use Kreait\Firebase\Exception\MessagingException;

class FCMService
{
    protected $messaging;

    public function __construct()
    {
        $factory = (new Factory)->withServiceAccount(config('firebase.credentials.file'));
        $this->messaging = $factory->createMessaging();
    }

    /**
     * Send notification to single device
     */
    public function sendToDevice(string $token, array $notification, array $data = [])
    {
        try {
            $message = CloudMessage::withTarget('token', $token)
                ->withNotification(Notification::create($notification['title'], $notification['body']))
                ->withData($data);

            return $this->messaging->send($message);
        } catch (MessagingException $e) {
            \Log::error('FCM Error: ' . $e->getMessage());
            return false;
        }
    }

    /**
     * Send to multiple devices
     */
    public function sendToMultipleDevices(array $tokens, array $notification, array $data = [])
    {
        try {
            $message = CloudMessage::new()
                ->withNotification(Notification::create($notification['title'], $notification['body']))
                ->withData($data);

            return $this->messaging->sendMulticast($message, $tokens);
        } catch (MessagingException $e) {
            \Log::error('FCM Multicast Error: ' . $e->getMessage());
            return false;
        }
    }

    /**
     * Send Android-specific notification with high priority
     */
    public function sendAndroidNotification(string $token, array $notification, array $data = [])
    {
        try {
            $androidConfig = AndroidConfig::fromArray([
                'priority' => 'high',
                'notification' => [
                    'title' => $notification['title'],
                    'body' => $notification['body'],
                    'sound' => 'default',
                    'click_action' => 'FLUTTER_NOTIFICATION_CLICK',
                ],
            ]);

            $message = CloudMessage::withTarget('token', $token)
                ->withNotification(Notification::create($notification['title'], $notification['body']))
                ->withAndroidConfig($androidConfig)
                ->withData($data);

            return $this->messaging->send($message);
        } catch (MessagingException $e) {
            \Log::error('Android FCM Error: ' . $e->getMessage());
            return false;
        }
    }

    /**
     * Send web push notification
     */
    public function sendWebNotification(string $token, array $notification, array $data = [])
    {
        try {
            $webConfig = WebPushConfig::fromArray([
                'notification' => [
                    'title' => $notification['title'],
                    'body' => $notification['body'],
                    'icon' => $notification['icon'] ?? '/icon.png',
                ],
                'fcm_options' => [
                    'link' => $notification['link'] ?? '/',
                ],
            ]);

            $message = CloudMessage::withTarget('token', $token)
                ->withNotification(Notification::create($notification['title'], $notification['body']))
                ->withWebPushConfig($webConfig)
                ->withData($data);

            return $this->messaging->send($message);
        } catch (MessagingException $e) {
            \Log::error('Web FCM Error: ' . $e->getMessage());
            return false;
        }
    }

    /**
     * Send data-only message (for background/silent notifications)
     */
    public function sendDataMessage(string $token, array $data)
    {
        try {
            $message = CloudMessage::withTarget('token', $token)
                ->withData($data);

            return $this->messaging->send($message);
        } catch (MessagingException $e) {
            \Log::error('FCM Data Message Error: ' . $e->getMessage());
            return false;
        }
    }

    /**
     * Subscribe tokens to topic
     */
    public function subscribeToTopic(array $tokens, string $topic)
    {
        try {
            return $this->messaging->subscribeToTopic($topic, $tokens);
        } catch (MessagingException $e) {
            \Log::error('Topic Subscription Error: ' . $e->getMessage());
            return false;
        }
    }

    /**
     * Send to topic
     */
    public function sendToTopic(string $topic, array $notification, array $data = [])
    {
        try {
            $message = CloudMessage::withTarget('topic', $topic)
                ->withNotification(Notification::create($notification['title'], $notification['body']))
                ->withData($data);

            return $this->messaging->send($message);
        } catch (MessagingException $e) {
            \Log::error('Topic Message Error: ' . $e->getMessage());
            return false;
        }
    }
}
```

---

## Step 6: Create API Controllers

### Notification Controller

```bash
php artisan make:controller Api/NotificationController
```

`app/Http/Controllers/Api/NotificationController.php`:

```php
namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Models\DeviceToken;
use App\Services\FCMService;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Validator;

class NotificationController extends Controller
{
    protected $fcmService;

    public function __construct(FCMService $fcmService)
    {
        $this->fcmService = $fcmService;
    }

    /**
     * Register device token
     */
    public function registerToken(Request $request)
    {
        $validator = Validator::make($request->all(), [
            'token' => 'required|string',
            'platform' => 'required|in:android,ios,web',
        ]);

        if ($validator->fails()) {
            return response()->json([
                'success' => false,
                'errors' => $validator->errors()
            ], 422);
        }

        $deviceToken = DeviceToken::updateOrCreate(
            [
                'user_id' => auth()->id(),
                'token' => $request->token,
            ],
            [
                'platform' => $request->platform,
                'is_active' => true,
            ]
        );

        return response()->json([
            'success' => true,
            'message' => 'Token registered successfully',
            'data' => $deviceToken
        ]);
    }

    /**
     * Send notification to authenticated user
     */
    public function sendToUser(Request $request)
    {
        $validator = Validator::make($request->all(), [
            'user_id' => 'required|exists:users,id',
            'title' => 'required|string|max:255',
            'body' => 'required|string',
            'data' => 'sometimes|array',
        ]);

        if ($validator->fails()) {
            return response()->json([
                'success' => false,
                'errors' => $validator->errors()
            ], 422);
        }

        $tokens = DeviceToken::where('user_id', $request->user_id)
            ->where('is_active', true)
            ->pluck('token')
            ->toArray();

        if (empty($tokens)) {
            return response()->json([
                'success' => false,
                'message' => 'No active tokens found for this user'
            ], 404);
        }

        $notification = [
            'title' => $request->title,
            'body' => $request->body,
        ];

        $data = $request->data ?? [];

        $result = $this->fcmService->sendToMultipleDevices($tokens, $notification, $data);

        return response()->json([
            'success' => true,
            'message' => 'Notification sent successfully',
            'result' => $result
        ]);
    }

    /**
     * Send notification to specific platform
     */
    public function sendByPlatform(Request $request)
    {
        $validator = Validator::make($request->all(), [
            'user_id' => 'required|exists:users,id',
            'platform' => 'required|in:android,ios,web',
            'title' => 'required|string|max:255',
            'body' => 'required|string',
            'data' => 'sometimes|array',
        ]);

        if ($validator->fails()) {
            return response()->json([
                'success' => false,
                'errors' => $validator->errors()
            ], 422);
        }

        $token = DeviceToken::where('user_id', $request->user_id)
            ->where('platform', $request->platform)
            ->where('is_active', true)
            ->value('token');

        if (!$token) {
            return response()->json([
                'success' => false,
                'message' => 'No active token found for this platform'
            ], 404);
        }

        $notification = [
            'title' => $request->title,
            'body' => $request->body,
        ];

        $data = $request->data ?? [];

        // Use platform-specific method
        $result = match($request->platform) {
            'android' => $this->fcmService->sendAndroidNotification($token, $notification, $data),
            'web' => $this->fcmService->sendWebNotification($token, $notification, $data),
            default => $this->fcmService->sendToDevice($token, $notification, $data),
        };

        return response()->json([
            'success' => true,
            'message' => 'Notification sent successfully',
            'result' => $result
        ]);
    }

    /**
     * Send background/silent notification
     */
    public function sendBackgroundNotification(Request $request)
    {
        $validator = Validator::make($request->all(), [
            'user_id' => 'required|exists:users,id',
            'data' => 'required|array',
        ]);

        if ($validator->fails()) {
            return response()->json([
                'success' => false,
                'errors' => $validator->errors()
            ], 422);
        }

        $tokens = DeviceToken::where('user_id', $request->user_id)
            ->where('is_active', true)
            ->pluck('token')
            ->toArray();

        if (empty($tokens)) {
            return response()->json([
                'success' => false,
                'message' => 'No active tokens found'
            ], 404);
        }

        $results = [];
        foreach ($tokens as $token) {
            $results[] = $this->fcmService->sendDataMessage($token, $request->data);
        }

        return response()->json([
            'success' => true,
            'message' => 'Background notifications sent',
            'results' => $results
        ]);
    }

    /**
     * Broadcast to all users
     */
    public function broadcast(Request $request)
    {
        $validator = Validator::make($request->all(), [
            'title' => 'required|string|max:255',
            'body' => 'required|string',
            'data' => 'sometimes|array',
        ]);

        if ($validator->fails()) {
            return response()->json([
                'success' => false,
                'errors' => $validator->errors()
            ], 422);
        }

        $tokens = DeviceToken::where('is_active', true)
            ->pluck('token')
            ->toArray();

        $notification = [
            'title' => $request->title,
            'body' => $request->body,
        ];

        $data = $request->data ?? [];

        // Send in batches of 500 (FCM limit)
        $chunks = array_chunk($tokens, 500);
        $results = [];

        foreach ($chunks as $chunk) {
            $results[] = $this->fcmService->sendToMultipleDevices($chunk, $notification, $data);
        }

        return response()->json([
            'success' => true,
            'message' => 'Broadcast sent successfully',
            'total_tokens' => count($tokens),
            'results' => $results
        ]);
    }

    /**
     * Remove device token
     */
    public function removeToken(Request $request)
    {
        $validator = Validator::make($request->all(), [
            'token' => 'required|string',
        ]);

        if ($validator->fails()) {
            return response()->json([
                'success' => false,
                'errors' => $validator->errors()
            ], 422);
        }

        DeviceToken::where('user_id', auth()->id())
            ->where('token', $request->token)
            ->delete();

        return response()->json([
            'success' => true,
            'message' => 'Token removed successfully'
        ]);
    }
}
```

---

## Step 7: API Routes

`routes/api.php`:

```php
use App\Http\Controllers\Api\NotificationController;
use Illuminate\Support\Facades\Route;

Route::middleware('auth:sanctum')->group(function () {
    // Device token management
    Route::post('/notifications/register-token', [NotificationController::class, 'registerToken']);
    Route::post('/notifications/remove-token', [NotificationController::class, 'removeToken']);
    
    // Send notifications (protected - admin only)
    Route::middleware('admin')->group(function () {
        Route::post('/notifications/send-to-user', [NotificationController::class, 'sendToUser']);
        Route::post('/notifications/send-by-platform', [NotificationController::class, 'sendByPlatform']);
        Route::post('/notifications/send-background', [NotificationController::class, 'sendBackgroundNotification']);
        Route::post('/notifications/broadcast', [NotificationController::class, 'broadcast']);
    });
});
```

---

## Step 8: Mobile Integration Examples

### Android (Kotlin/Java)

Add to `build.gradle`:

```gradle
implementation 'com.google.firebase:firebase-messaging:23.3.1'
```

Kotlin Example:

```kotlin
// Get FCM token
FirebaseMessaging.getInstance().token.addOnCompleteListener { task ->
    if (task.isSuccessful) {
        val token = task.result
        // Send to Laravel API
        sendTokenToServer(token, "android")
    }
}

// Handle foreground messages
class MyFirebaseMessagingService : FirebaseMessagingService() {
    override fun onMessageReceived(remoteMessage: RemoteMessage) {
        remoteMessage.notification?.let {
            showNotification(it.title, it.body)
        }
    }
}
```

### iOS (Swift)

```swift
import FirebaseMessaging

// Get FCM token
Messaging.messaging().token { token, error in
    if let token = token {
        // Send to Laravel API
        sendTokenToServer(token: token, platform: "ios")
    }
}

// Handle notifications
extension AppDelegate: UNUserNotificationCenterDelegate {
    func userNotificationCenter(_ center: UNUserNotificationCenter,
                              willPresent notification: UNNotification) {
        // Handle foreground notification
    }
}
```

---

## Step 9: Web Integration

### HTML + JavaScript

```html
<!DOCTYPE html>
<html>
<head>
    <title>FCM Web Notifications</title>
</head>
<body>
    <button onclick="requestPermission()">Enable Notifications</button>

    <script src="https://www.gstatic.com/firebasejs/10.7.0/firebase-app-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/10.7.0/firebase-messaging-compat.js"></script>

    <script>
        // Firebase config
        const firebaseConfig = {
            apiKey: "YOUR_API_KEY",
            authDomain: "YOUR_PROJECT.firebaseapp.com",
            projectId: "YOUR_PROJECT_ID",
            storageBucket: "YOUR_PROJECT.appspot.com",
            messagingSenderId: "YOUR_SENDER_ID",
            appId: "YOUR_APP_ID"
        };

        firebase.initializeApp(firebaseConfig);
        const messaging = firebase.messaging();

        function requestPermission() {
            Notification.requestPermission().then((permission) => {
                if (permission === 'granted') {
                    getToken();
                }
            });
        }

        function getToken() {
            messaging.getToken({
                vapidKey: 'YOUR_VAPID_KEY'
            }).then((token) => {
                console.log('Token:', token);
                sendTokenToServer(token);
            });
        }

        function sendTokenToServer(token) {
            fetch('/api/notifications/register-token', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                    'Authorization': 'Bearer ' + YOUR_AUTH_TOKEN
                },
                body: JSON.stringify({
                    token: token,
                    platform: 'web'
                })
            });
        }

        // Handle foreground messages
        messaging.onMessage((payload) => {
            console.log('Message received:', payload);
            new Notification(payload.notification.title, {
                body: payload.notification.body,
                icon: '/icon.png'
            });
        });
    </script>
</body>
</html>
```

### Firebase Service Worker

Create `public/firebase-messaging-sw.js`:

```javascript
importScripts('https://www.gstatic.com/firebasejs/10.7.0/firebase-app-compat.js');
importScripts('https://www.gstatic.com/firebasejs/10.7.0/firebase-messaging-compat.js');

firebase.initializeApp({
    apiKey: "YOUR_API_KEY",
    authDomain: "YOUR_PROJECT.firebaseapp.com",
    projectId: "YOUR_PROJECT_ID",
    storageBucket: "YOUR_PROJECT.appspot.com",
    messagingSenderId: "YOUR_SENDER_ID",
    appId: "YOUR_APP_ID"
});

const messaging = firebase.messaging();

// Handle background messages
messaging.onBackgroundMessage((payload) => {
    console.log('Background message:', payload);
    
    const notificationTitle = payload.notification.title;
    const notificationOptions = {
        body: payload.notification.body,
        icon: '/icon.png'
    };

    self.registration.showNotification(notificationTitle, notificationOptions);
});
```

---

## Step 10: Testing Examples

### Using Postman/cURL

**Register Token:**

```bash
curl -X POST http://your-app.com/api/notifications/register-token \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "token": "FCM_DEVICE_TOKEN",
    "platform": "android"
  }'
```

**Send Notification:**

```bash
curl -X POST http://your-app.com/api/notifications/send-to-user \
  -H "Authorization: Bearer YOUR_ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": 1,
    "title": "Test Notification",
    "body": "This is a test message",
    "data": {
      "type": "message",
      "id": "123"
    }
  }'
```

**Send Background Notification:**

```bash
curl -X POST http://your-app.com/api/notifications/send-background \
  -H "Authorization: Bearer YOUR_ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": 1,
    "data": {
      "action": "sync",
      "timestamp": "2025-01-02T10:00:00Z"
    }
  }'
```

---

## Important Notes

1. **Security**: Never expose your Firebase server key publicly
2. **Token Management**: Regularly clean up inactive tokens
3. **Rate Limits**: FCM has rate limits - batch your requests
4. **Error Handling**: Always handle FCM exceptions properly
5. **Testing**: Test on real devices for accurate results
6. **VAPID Key**: For web, generate VAPID key in Firebase Console
7. **Background Restrictions**: iOS has strict background notification rules

## Troubleshooting

- **Token not received**: Check Firebase console permissions
- **Notifications not showing**: Verify app is in foreground/background
- **Web notifications**: Ensure HTTPS and service worker registered
- **Android priority**: Use `priority: high` for immediate delivery
