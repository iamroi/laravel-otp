# Laravel OTP(One-Time Password)

## Introduction

Most of the web applications need an OTP(one-time password) or secure code to validate their users. This package allows
you to send/resend and validate OTP for users authentication with user-friendly methods.

## Basic Usage:

```php
<?php

// Send an OTP to a valid mobile number.
OTP()->send('+989389599530');

OTP('+989389599530');

// Send otp via channels
OTP()->channel(['otp_sms', 'mail', \App\Channels\CustomSMSChannel::class])
     ->send('+989389599530');

OTP('+989389599530', ['otp_sms', 'mail', \App\Channels\CustomSMSChannel::class]);

// Send otp for specific user provider
OTP()->useProvider('admins')
     ->send('+989389599530');

// Validate OTP
OTP()->validate('+989389599530', 'token_123');

OTP('+989389599530', 'token_123');

OTP()->useProvider('users')
     ->validate('+989389599530', 'token_123');
```

## Installation

You can install the package via composer:

```shell
composer require fouladgar/laravel-otp
```

## Configuration

To get started, you should publish the `config/otp.php` config file with:

```
php artisan vendor:publish --provider="Fouladgar\OTP\ServiceProvider" --tag="config"
```

### Token Storage

After generating a token, we need to store it in a storage. This package supports two drivers: `cache` and `database`
which the default driver is `cache`. You may specify which storage driver you would like to be used for saving tokens in
your application:

```php
// config/otp.php

<?php

return [
    /**
    |Supported drivers: "cache", "database"
    */
    'token_storage' => 'cache',
];
```

##### Cache

When using the `cache` driver, the token will be stored in a cache driver configured by your application. In this case,
your application performance is more than when using database definitely.

##### Database

It means after migrating, a table will be created which your application needs to store verification tokens.

> If you’re using another column name for `mobile` phone or even `otp_tokens` table, you can customize their values in config file:

```php
// config/otp.php

<?php

return [

    'mobile_column' => 'mobile',

    'token_table'   => 'otp_token',

    //...
];

```

Depending on the `token_storage` config, the package will create a token table. Also, a `mobile` column will be added to
your `users` ([default provider](#user-providers)) table to show user verification state and store user's mobile phone.

All right! Now you should migrate the database:

```
php artisan migrate
```

> **Note:** When you are using OTP to login user, consider all columns must be nullable except for the `mobile` column. Because, after verifying OTP, a user record will be created if the user does not exist.

## User providers

You may wish use the OTP for variant users. Laravel OTP allows you to define and manage many user providers that you
need. In order to set up, you should open `config/otp.php` file and define your providers:

```php
// config/otp.php

<?php

return [
    //...

    'default_provider' => 'users',

    'user_providers'  => [
        'users' => [
            'table'      => 'users',
            'model'      => \App\Models\User::class, // if Laravel < 8, change it to \App\User::class
            'repository' => \Fouladgar\OTP\NotifiableRepository::class,
        ],

       // 'admins' => [
       //   'model'      => \App\Models\Admin::class,
       //   'repository' => \Fouladgar\OTP\NotifiableRepository::class,
       // ],
    ],

    //...
];
```

> **Note:** You may also change the default repository and replace your own repository. But every repository must implement `Fouladgar\OTP\Contracts\NotifiableRepositoryInterface` interface

#### Model Preparation

Every model must implement `Fouladgar\OTP\Contracts\OTPNotifiable` and also use
this `Fouladgar\OTP\Concerns\HasOTPNotify` trait:

```php
<?php

namespace App\Models;

use Fouladgar\OTP\Concerns\HasOTPNotify;
use Fouladgar\OTP\Contracts\OTPNotifiable;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;

class User extends Authenticatable implements OTPNotifiable
{
    use Notifiable;
    use HasOTPNotify;

    // ...
}
```

### SMS Client

You can use any SMS services for sending verification messages(it depends on your choice). For sending notifications via
this package, first you need to implement the `Fouladgar\OTP\Contracts\SMSClient` contract. This contract requires you
to implement `sendMessage` method.

This method will return your SMS service API results via a `Fouladgar\OTP\Notifications\Messages\MessagePayload` object
which contains user **mobile** and **token** message:

```php
<?php

namespace App;

use Fouladgar\OTP\Contracts\SMSClient;
use Fouladgar\OTP\Notifications\Messages\MessagePayload;

class SampleSMSClient implements SMSClient
{
    protected $SMSService;

    public function sendMessage(MessagePayload $payload)
    {
        // preparing SMSService ...

        return $this->SMSService
                 ->send($payload->to(), $payload->content());
    }

    // ...
}
```

> In above example, `SMSService` can be replaced with your chosen SMS service along with its respective method.

Next, you should set the your `SampleSMSClient` class in config file:

```php
// config/otp.php

<?php

return [

  'sms_client' => \App\SampleSMSClient::class,

  //...
];
```

## Practical Example

Here we prepared a practical example. Suppose you are going to login/register a customer by sending an OTP:

```php
<?php

namespace App\Http\Controllers;

use App\Models\User;
use Fouladgar\OTP\Exceptions\InvalidOTPTokenException;
use Fouladgar\OTP\OTPBroker as OTPService;
use Illuminate\Http\Request;
use Throwable;

class AuthController
{
    /**
    * @var OTPService
    */
    private $OTPService;

    public function __construct(OTPService $OTPService)
    {
        $this->OTPService = $OTPService;
    }

    public function sendOTP(Request $request)
    {
        try {
            /** @var User $customer */
            $customer = $this->OTPService->send($request->get('mobile'));
        } catch (Throwable $ex) {
          // or prepare and return a view.
           return response()->json(['message'=>'An Occurred unexpected error.'], 500);
        }

        return response()->json(['message'=>'A token sent to:'. $customer->mobile]);
    }

    public function verifyOTPAndLogin(Request $request)
    {
        try {
            /** @var User $customer */
            $customer = $this->OTPService->validate($request->get('mobile'), $request->get('token'));

            // and do login actions...

        } catch (InvalidOTPTokenException $exception){
             return response()->json(['error'=>$exception->getMessage()],$exception->getCode());
        } catch (Throwable $ex) {
            return response()->json(['message'=>'An Occurred unexpected error.'], 500);
        }

         return response()->json(['message'=>'Login has been successfully.']);
    }
}

```

## Customization

### Notification Default Channel Customization

For sending OTP notification there is a default channel. But this package allows you to use your own notification
channel. In order to replace, you should specify channel class here:

```php
//config/otp.php
<?php
return [
    // ...

    'channel' => \Fouladgar\OTP\Notifications\Channels\OTPSMSChannel::class,
];
```

> **Note:** If you change the default sms channel, the `sms_client` will be an optional config. Otherwise, you must define your sms client.

### Notification SMS and Email Customization

OTP notification prepares a default sms and email format that are satisfied for most application. However, you can
customize how the mail/sms message is constructed.

To get started, pass a closure to the toSMSUsing/toMailUsing method provided by
the `Fouladgar\OTP\Notifications\OTPNotification` notification. The closure will receive the notifiable model instance
that is receiving the notification as well as the `token` for validating. Typically, you should call the toMailUsing
method from the boot method of your application's `App\Providers\AuthServiceProvider` class:

```php
<?php

use Fouladgar\OTP\Notifications\OTPNotification;
use Fouladgar\OTP\Notifications\Messages\OTPMessage;
use Illuminate\Notifications\Messages\MailMessage;

public function boot()
{
    // ...

    // SMS Customization
    OTPNotification::toSMSUsing(function($notifiable, $token) {
        return (new OTPMessage())
                    ->to($notifiable->mobile)
                    ->content('Your OTP Token is: '.$token);
    });

    //Email Customization
    OTPNotification::toMailUsing(function ($notifiable, $token) {
        return (new MailMessage)
            ->subject('OTP Request')
            ->line('Your OTP Token is: '.$token);
    });
}
```

## Translates

To publish translation file you may use this command:

```
php artisan vendor:publish --provider="Fouladgar\OTP\ServiceProvider" --tag="lang"
```

you can customize in provided language file:

```php
// resources/lang/vendor/OTP/en/otp.php

<?php

return [
    'otp_token' => 'Your OTP Token is: :token.',

    'otp_subject' => 'OTP request',
];
```

## Testing

```sh
composer test
```

## Changelog

Please see [CHANGELOG](CHANGELOG.md) for more information what has changed recently.

## Contributing

Please see [CONTRIBUTING](CONTRIBUTING.md) for details.

## Security

If you discover any security related issues, please email fouladgar.dev@gmail.com instead of using the issue tracker.

## License

Laravel-OTP is released under the MIT License. See the bundled
[LICENSE](https://github.com/mohammad-fouladgar/laravel-mobile-verification/blob/master/LICENSE)
file for details.

Built with :heart: for you.
