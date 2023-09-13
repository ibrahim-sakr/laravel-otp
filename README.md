# Laravel OTP

## Introduction

OTP Package for Laravel using class based system. Every Otp is a class that does something. For example, an `EmailVerificationOtp` which will mark the account as verified.

## Installation

Install via composer

```bash
composer require sadiqsalau/laravel-otp
```

Publish config file

```bash
php artisan vendor:publish --provider="SadiqSalau\LaravelOtp\OtpServiceProvider"
```

## Usage

### Generate OTP

```bash
php artisan make:otp {name}
```

A new Otp class will be generated into the `app/Otp` directory. e.g

```bash
php artisan make:otp UserRegistrationOtp
```

Every Otp must implement the `process` method which will be called after verification. There the Otp can perform the necessary action and return any result.

```php
<?php

namespace App\Otp;

use SadiqSalau\LaravelOtp\Contracts\OtpInterface as Otp;

use Illuminate\Auth\Events\Registered;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Hash;

use App\Models\User;

class UserRegistrationOtp implements Otp
{
    /**
     * Constructs Otp class
     */
    public function __construct(
        public string $name,
        public string $email,
        public string $password
    ) {
        //
    }

    /**
     * Processes the Otp
     *
     * @return User
     */
    public function process()
    {
        /** @var User */
        $user = User::unguarded(function () {
            return User::create([
                'name'                  => $this->name,
                'email'                 => $this->email,
                'password'              => Hash::make($this->password),
                'email_verified_at'     => now(),
            ]);
        });

        event(new Registered($user));

        Auth::login($user);

        return $user;
    }
}

```

### Sending OTP

```php
<?php
use SadiqSalau\LaravelOtp\Facades\Otp;

Otp::identifier($identifier)->send($otp, $notifiable);
```

- `$otp`: The otp to send.
- `$notifiable`: AnonymousNotifiable or Notifiable instance.

```php
use Illuminate\Support\Facades\Notification;
use SadiqSalau\LaravelOtp\Facades\Otp;
use App\Otp\UserRegistrationOtp;

Route::post('/register', function(Request $request){
    //...
    return Otp::identifier($request->email)->send(
        new UserRegistrationOtp(
            name: $request->name,
            email: $request->email,
            password: $request->password
        ),
        Notification::route('mail', $request->email)
    );
});
```

Returns

```php
['status' => Otp::OTP_SENT] // Success: otp.sent
```

### Verify OTP

```php
<?php
use SadiqSalau\LaravelOtp\Facades\Otp;

Otp::identifier($identifier)->attempt($code);
```

- `$code`: The otp code to compare against.

Returns

```php
['status' => Otp::OTP_EMPTY]        // Error: otp.empty
['status' => Otp::OTP_MISMATCHED]  // Error: otp.mismatched
['status' => Otp::OTP_PROCESSED, 'result'=>[]] // Success: otp.processed
```

The `result` key contains the returned value of the `process` method of the Otp class

```php
use SadiqSalau\LaravelOtp\Facades\Otp;

Route::get('/otp/verify', function (Request $request) {
    //...

    $otp = Otp::identifier($request->email)->attempt($request->code);

    if($otp['status'] != Otp::OTP_PROCESSED)
    {
        return __($otp['status']);
    }
    else {
        return $otp['result'];
    }
});
```

### Resend OTP

```php
<?php
use SadiqSalau\LaravelOtp\Facades\Otp;

Otp::identifier($identifier)->update();
```

Returns

```php
['status' => Otp::OTP_EMPTY]    // Error: otp.empty
['status' => Otp::OTP_SENT]     // Success: otp.sent
```

```php
use SadiqSalau\LaravelOtp\Facades\Otp;

Route::get('/otp/resend', function () {
    return Otp::update();
});
```

### Setting Identifier

The cache store uses the request's IP address to uniquely identify the Otp, you may change this by calling the `identifier` method of the Otp facade.

```php
<?php
use SadiqSalau\LaravelOtp\Facades\Otp;

Otp::identifier($request->email)->send(...);
```

```php
<?php
use SadiqSalau\LaravelOtp\Facades\Otp;

Otp::identifier($identifier)->send(...);
Otp::identifier($identifier)->attempt(...);
Otp::identifier($identifier)->update();
```

## Config

Config file can be found at `config/otp.php` after publishing the package

- `store` - The store is a class for storing the Otp. The package provides two stores by default. All stores must implement `SadiqSalau\LaravelOtp\Contracts\OtpStoreInterface`. The default store is the `CacheStore`

```php
use SadiqSalau\LaravelOtp\Stores\CacheStore;
use SadiqSalau\LaravelOtp\Stores\SessionStore;

//...
'store' => CacheStore::class
```

- `store_key` - Key used by the store to retrieve the Otp
- `format` - Format of generated Otp code (`numeric` | `alphanumeric` | `alpha`)
- `length` - Length of generated Otp code
- `expires` - Number of minutes before Otp expires,
- `notification` - Custom notification class to use, default is `SadiqSalau\LaravelOtp\OtpNotification`

## Translations

The package doesn't provide translations out of the box, but here is an example.
Create a new translation file: `lang/en/otp.php`

```php
<?php

return [

    /*
    |--------------------------------------------------------------------------
    | OTP Language Lines
    |--------------------------------------------------------------------------
    |
    | The following language lines are used by the OTP broker
    |
    */

    'sent'          => 'We have sent your OTP code!',
    'empty'         => 'No OTP!',
    'expired'       => 'Expired OTP!',
    'mismatched'    => 'Mismatched OTP code!',
    'processed'     => 'OTP was successfully processed!'
];

```

Then translate the status

```php
return __($otp['status'])
```

## API

- `Otp::send(OtpInterface $otp, mixed $notifiable)` - Send Otp to a notifiable

- `Otp::attempt(string $code)` - Attempt Otp code, returns the result of calling the `process` method of the Otp

- `Otp::update()` - Resend and update current Otp

- `Otp::clear()` - Clear Otp from store

- `Otp::identifier(mixed $identifier)` - Override identifier of Otp store

- `Otp::useGenerator(callable $callback)` - Set custom generator to use, generator will be called with `$format` and `$length`

- `Otp::generateOtpCode($format, $length)` - Generates the Otp code

## Contribution

Contributions are welcomed.
