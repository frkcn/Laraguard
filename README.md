<p align="center">
  <img src="logo.jpg">
</p>

[![Latest Version on Packagist](https://img.shields.io/packagist/v/darkghosthunter/laraguard.svg?style=flat-square)](https://packagist.org/packages/darkghosthunter/laraguard) [![License](https://poser.pugx.org/darkghosthunter/laraguard/license)](https://packagist.org/packages/darkghosthunter/laraguard)
![](https://img.shields.io/packagist/php-v/darkghosthunter/laraguard.svg)
![](https://github.com/DarkGhostHunter/Laraguard/workflows/PHP%20Composer/badge.svg)
[![Coverage Status](https://coveralls.io/repos/github/DarkGhostHunter/Laraguard/badge.svg?branch=master)](https://coveralls.io/github/DarkGhostHunter/Laraguard?branch=master)

# Laraguard

Two-Factor Authentication via TOTP for all your users out-of-the-box.

This package enables authentication using 6 digits codes. No need for external APIs.

## Requirements

* [Laravel 8.39 or later](https://github.com/laravel/framework/blob/8.x/CHANGELOG-8.x.md#v8390-2021-04-27)
* PHP 8.0 or later.

> For older versions support, consider helping by sponsoring or donating.

## Installation

Fire up Composer and require this package in your project.

    composer require darkghosthunter/laraguard

That's it.

### How this works

This package adds a **Contract** to detect if, after the credentials are deemed valid, should use Two-Factor Authentication as a second layer of authentication.

It includes a custom **view** and a **callback** to handle the Two-Factor authentication itself during login attempts.

Works without middleware or new guards, but you can go full manual if you want.

## Usage

First, create the `two_factor_authentications` table by publishing the migration and migrating:

    php artisan vendor:publish --provider="DarkGhostHunter\Laraguard\LaraguardServiceProvider" --tag="migrations"
    php artisan migrate

This will create a table to handle the Two-Factor Authentication information for each model you want to attach to 2FA.

> If you're [upgrading from 3.0](UPGRADE.md), you should run a special migration.

Add the `TwoFactorAuthenticatable` _contract_ and the `TwoFactorAuthentication` trait to the User model, or any other model you want to make Two-Factor Authentication available. 

```php
<?php

namespace App;

use Illuminate\Foundation\Auth\User as Authenticatable;
use DarkGhostHunter\Laraguard\TwoFactorAuthentication;
use DarkGhostHunter\Laraguard\Contracts\TwoFactorAuthenticatable;

class User extends Authenticatable implements TwoFactorAuthenticatable
{
    use TwoFactorAuthentication;
    
    // ...
}
```

The contract is used to identify the model using Two-Factor Authentication, while the trait conveniently implements the methods required to handle it.

### Enabling Two-Factor Authentication

To enable Two-Factor Authentication successfully, the User must sync the Shared Secret between its Authenticator app and the application. 

> Some free Authenticator Apps are [iOS Authenticator](https://www.apple.com/ios/ios-15-preview/features/#:~:text=Built-in%20authenticator), [FreeOTP](https://freeotp.github.io/), [Authy](https://authy.com/), [andOTP](https://github.com/andOTP/andOTP), [Google](https://apps.apple.com/app/google-authenticator/id388497605) [Authenticator](https://play.google.com/store/apps/details?id=com.google.android.apps.authenticator2&hl=en), and [Microsoft Authenticator](https://www.microsoft.com/en-us/account/authenticator), to name a few.

To start, generate the needed data using the `createTwoFactorAuth()` method. Once you do, you can show the Shared Secret to the User as a string or QR Code (encoded as SVG) in your view.

```php
use Illuminate\Http\Request;

public function prepareTwoFactor(Request $request)
{
    $secret = $request->user()->createTwoFactorAuth();
    
    return view('user.2fa', [
        'as_qr_code' => $secret->toQr(),     // As QR Code
        'as_uri'     => $secret->toUri(),    // As "otpauth://" URI.
        'as_string'  => $secret->toString(), // As a string
    ]);
}
```

> When you use `createTwoFactorAuth()` on someone with Two-Factor Authentication already enabled, the previous data becomes permanently invalid. This ensures a User **never** has two Shared Secrets enabled at any given time.

Then, the User must confirm the Shared Secret with a Code generated by their Authenticator app. The `confirmTwoFactorAuth()` method will automatically enable it if the code is valid.

```php
use Illuminate\Http\Request;

public function confirmTwoFactor(Request $request)
{
    $request->validate([
        'code' => 'required|numeric'
    ]);
    
    $activated = $request->user()->confirmTwoFactorAuth($request->code);
    
    // ...
}
```

If the User doesn't issue the correct Code, the method will return `false`. You can tell the User to double-check its device's timezone, or create another Shared Secret with `createTwoFactorAuth()`.

### Recovery Codes

Recovery Codes are automatically generated each time the Two-Factor Authentication is enabled. By default, a Collection of ten one-use 8-characters codes are created.

You can show them using `getRecoveryCodes()`.

```php
use Illuminate\Http\Request;

public function confirmTwoFactor(Request $request)
{
    if ($request->user()->confirmTwoFactorAuth($request->code)) {
        return $request->user()->getRecoveryCodes();
    } else {
        return 'Try again!';
    }
}
```

You're free on how to show these codes to the User, but **ensure** you show them one time after a successfully enabling Two-Factor Authentication, and ask him to print them somewhere.

> These Recovery Codes are handled automatically when the User sends it instead of a TOTP code. If it's a recovery code, the package will use and mark it as invalid.

The User can generate a fresh batch of codes using `generateRecoveryCodes()`, which automatically invalidates the previous batch.

```php
use Illuminate\Http\Request;

public function showRecoveryCodes(Request $request)
{
    return $request->user()->generateRecoveryCodes();
}
```

> If the User depletes his recovery codes without disabling Two-Factor Authentication, or Recovery Codes are deactivated, **he may be locked out forever without his Authenticator app**. Ensure you have countermeasures in these cases.

### Logging in

To login, the user must issue a TOTP code along their credentials. Simply use `attemptWhen()` with Laraguard, which will automatically do the checks for you. By default, it checks for the `2fa_code` input name, but you can issue your own.

```php
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use DarkGhostHunter\Laraguard\Laraguard;

public function login(Request $request)
{
    // ...
    
    $credentials = $request->only('email', 'password');
    
    if (Auth::attemptWhen($credentials, Laraguard::hasCode(), $request->filled('remember'))) {
        return redirect()->home(); 
    }
    
    return back()->withErrors(['email' => 'Bad credentials'])
}
```

Behind the scenes, once the User is retrieved and validated from your guard of choice, it makes an additional check for a valid TOTP code. If it's invalid, it will return false and no authentication will happen.

> For Laravel Breeze, you may need to edit the `LoginRequest::authenticate()` call.
> For Laravel Fortify and Jetstream, you may need to set a custom callback with the `Fortify::authenticateUsing()` method.

#### Separating the TOTP requirement

In some occasions you will want to tell the user the authentication failed not because the credentials were incorrect, but because of the TOTP code was invalid.

You can use the `hasCodeOrFails()` method that does the same, but throws a validation exception, which is handled gracefully by the framework. It even accepts a custom message in case of failure, otherwise a default [translation](#translations) line will be used.

```php
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use DarkGhostHunter\Laraguard\Laraguard;

public function login(Request $request)
{
    // ...
    
    $credentials = $request->only('email', 'password');
    
    if (Auth::attemptWhen($credentials, Laraguard::hasCodeOrFails(), $request->filled('remember'))) {
        return redirect()->home(); 
    }
    
    return back()->withErrors(['email', 'Authentication failed!']);
}
```

Since it's a `ValidationException`, you can catch it and do more complex things, like those fancy views that hold the login procedure until the correct TOTP code is issued. 

### Deactivation

You can deactivate Two-Factor Authentication for a given User using the `disableTwoFactorAuth()` method. This will automatically invalidate the authentication data, allowing the User to log in with just his credentials.

```php
public function disableTwoFactorAuth(Request $request)
{
    $request->user()->disableTwoFactorAuth();
    
    return 'Two-Factor Authentication has been disabled!';
}
```

## Events

The following events are fired in addition to the default Authentication events.

* `TwoFactorEnabled`: An User has enabled Two-Factor Authentication.
* `TwoFactorRecoveryCodesDepleted`: An User has used his last Recovery Code.
* `TwoFactorRecoveryCodesGenerated`: An User has generated a new set of Recovery Codes.
* `TwoFactorDisabled`: An User has disabled Two-Factor Authentication.

> You can use `TwoFactorRecoveryCodesDepleted` to tell the User to create more Recovery Codes or mail them some more.

## Middleware

Laraguard comes with two middleware for your routes: `2fa.enabled` and `2fa.confirm`.

> To avoid unexpected results, middleware only act on your users models implementing the `TwoFactorAuthenticatable` contract. If a user model doesn't implement it, the middleware will bypass any 2FA logic.

### Require 2FA

If you need to ensure the User has Two-Factor Authentication enabled before entering a given route, you can use the `2fa.enabled` middleware. Users who implement the `TwoFactorAuthenticatable` contract and have 2FA disabled will be redirected to a route name containing the warning, which is `2fa.notice` by default.

```php
Route::get('system/settings')
    ->uses('SystemSettingsController@show')
    ->middleware('2fa.enabled');
```

You can implement the view easily with the one included in this package, optionally with a URL to point the user to enable 2FA:

```php
use Illuminate\Support\Facades\Route;

Route::view('2fa-required', 'laraguard::notice', [
    'url' => url('settings/2fa')
])->name('2fa.notice');
```

Alternatively, you can just redirect the user to the named route where he can enable 2FA.

```php
use Illuminate\Support\Facades\Route

Route::get('system/settings')
    ->uses('SystemSettingsController@show')
    ->middleware('2fa.enabled:settings.2fa');
```

### Confirm 2FA

Much like the [`password.confirm` middleware](https://laravel.com/docs/authentication#password-confirmation), you can also ask the user to confirm an action using `2fa.confirm` if it has Two-Factor Authentication enabled.

```php
Route::get('api/token')
    ->uses('ApiTokenController@show')
    ->middleware('2fa.confirm');
```

Since a user without 2FA enabled won't be asked for a code, you use it with `2fa.require` to enforce it.

```php
Route::get('api/token')
    ->uses('ApiTokenController@show')
    ->middleware('2fa.require', '2fa.confirm');
```

Laraguard uses its [`Confirm2FACodeController`](src/Http/Controllers/Confirm2FACodeController.php) to handle the form view. [You can point your own controller actions](#confirmation-middleware). The [`Confirms2FACode`](src/Http/Controllers/Confirms2FACode.php) trait will aid you in not reinventing the wheel.

## Validation

Sometimes you may want to manually trigger a TOTP validation in any part of your application for the authenticated user. You can validate a TOTP code for the authenticated user using the `totp_code` rule.

```php
public function checkTotp(Request $request)
{
    $request->validate([
        'code' => 'required|totp_code'
    ]);

    // ...
}
```

This rule will succeed if the user is authenticated, it has Two-Factor Authentication enabled, and the code is correct.

## Translations

Laraguard comes with translation files (only for english) that you can use immediately in your application. These are also used for the [validation rule](#validation).

```php
public function disableTwoFactorAuth()
{
    // ...

    session()->flash('2fa_disabled', trans('laraguard::messages.disabled'));

    return back();
}
```

To add your own in your language, publish the translation files. These will be located in `resources/vendor/laraguard`:

    php artisan vendor:publish --provider="DarkGhostHunter\Laraguard\LaraguardServiceProvider" --tag="translations"

## Configuration

To further configure the package, publish the configuration files and assets:

    php artisan vendor:publish --provider="DarkGhostHunter\Laraguard\LaraguardServiceProvider"

You will receive the `config/laraguard.php` config file with the following contents:

```php
return [
    'model' => \DarkGhostHunter\Laraguard\Eloquent\TwoFactorAuthentication::class,
    'cache' => [
        'store' => null,
        'prefix' => '2fa.code'
    ],
    'recovery' => [
        'enabled' => true,
        'codes' => 10,
        'length' => 8,
	],
    'safe_devices' => [
        'enabled' => false,
        'max_devices' => 3,
        'expiration_days' => 14,
	],
    'confirm' => [
        'timeout' => 10800,
        'view' => 'DarkGhostHunter\Laraguard\Http\Controllers\Confirm2FACodeController@showConfirmForm',
        'action' => 'DarkGhostHunter\Laraguard\Http\Controllers\Confirm2FACodeController@confirm'
    ],
    'secret_length' => 20,
    'issuer' => env('OTP_TOTP_ISSUER'),
    'totp' => [
        'digits' => 6,
        'seconds' => 30,
        'window' => 1,
        'algorithm' => 'sha1',
    ],
    'qr_code' => [
        'size' => 400,
        'margin' => 4
    ],
];
```

### Eloquent Model

```php
return [
    'model' => \DarkGhostHunter\Laraguard\Eloquent\TwoFactorAuthentication::class,
];
```

This is the model where the data for Two-Factor Authentication is saved, like the shared secret and recovery codes, and associated to the models implementing `TwoFactorAuthenticatable`.

You can change this model for your own if you wish, as long it implements the `TwoFactorTotp` contract.

### Cache Store

```php
return  [
    'cache' => [
        'store' => null,
        'prefix' => '2fa.code'
    ],
];
```

[RFC 6238](https://tools.ietf.org/html/rfc6238#section-5) states that one-time passwords shouldn't be able to be usable more than once, even if is still inside the time window. For this, we need to use the Cache to save the code for a given period.

You can change the store to use, which it's the default used by your application, and the prefix to use as cache keys, in case of collisions.

### Recovery

```php
return [
    'recovery' => [
        'enabled' => true,
        'codes' => 10,
        'length' => 8,
    ],
];
```

Recovery codes handling are enabled by default, but you can disable it. If you do, ensure Users can authenticate by other means, like sending an email with a link to a signed URL that logs him in and disables Two-Factor Authentication, or SMS.

The number and length of codes generated is configurable. 10 Codes of 8 random characters are enough for most authentication scenarios.

### Safe devices

```php
return [
    'safe_devices' => [
        'enabled' => false,
        'max_devices' => 3,
        'expiration_days' => 14,
    ],
];
```

Enabling this option will allow the application to "remember" a device using a cookie, allowing it to bypass Two-Factor Authentication once a code is verified in that device. When the User logs in again in that device, it won't be prompted for a 2FA Code again.

There is a limit of devices that can be saved, but usually three is enough (phone, tablet and PC). New devices will displace the oldest devices registered. Devices are considered no longer "safe" until a set amount of days.

You can change the maximum number of devices saved and the amount of days of validity once they're registered. More devices and more expiration days will make the Two-Factor Authentication less secure.

> When re-enabling Two-Factor Authentication, the list of devices is automatically invalidated.

### Confirmation Middleware

```php
return [
    'confirm' => [
        'timeout' => 10800, // 3 hours
        'view' => 'DarkGhostHunter\Laraguard\Http\Controllers\Confirm2FACodeController@showConfirmForm',
        'action' => 'DarkGhostHunter\Laraguard\Http\Controllers\Confirm2FACodeController@confirm'
    ],
];
```

If the `view` or `action` are not `null`, the `2fa/notice` and `2fa/confirm` routes will be registered to handle 2FA code notice and confirmation for the [`2fa.confirm` middleware](#confirm-2fa). If you disable it, you will have to register the routes and controller actions yourself.

This array sets:

- By how much to "remember" the 2FA Code confirmation.
- The action that shows the 2FA Code form.
- The action that receives the 2FA Code and validates it.

### Secret length

```php
return [
    'secret_length' => 20,
];
```

This controls the length (in bytes) used to create the Shared Secret. While a 160-bit shared secret is enough, you can tighten or loosen the secret length to your liking.

It's recommended to use 128-bit or 160-bit because some Authenticator apps may have problems with non-RFC-recommended lengths.

### TOTP Configuration 

```php
return [
    'issuer' => env('OTP_TOTP_ISSUER'),
    'totp' => [
        'digits' => 6,
        'seconds' => 30,
        'window' => 1,
        'algorithm' => 'sha1',
    ],
];
```

This controls TOTP code generation and verification mechanisms:

* Issuer: The name of the issuer of the TOTP. Default is the application name. 
* TOTP Digits: The amount of digits to ask for TOTP code. 
* TOTP Seconds: The number of seconds a code is considered valid.
* TOTP Window: Additional steps of seconds to keep a code as valid.
* TOTP Algorithm: The system-supported algorithm to handle code generation.

This configuration values are always passed down to the authentication app as URI parameters:

    otpauth://totp/Laravel:taylor@laravel.com?secret=THISISMYSECRETPLEASEDONOTSHAREIT&issuer=Laravel&label=taylor%40laravel.com&algorithm=SHA1&digits=6&period=30

These values are printed to each 2FA data record inside the application. Changes will only take effect for new activations.

> Do not edit these parameters if you plan to use publicly available Authenticator apps, since some of them **may not support non-standard configuration**, like more digits, different period of seconds or other algorithms.

### QR Code Configuration 

```php
return [
    'qr_code' => [
        'size' => 400,
        'margin' => 4
    ],
];
```

This controls the size and margin used to create the QR Code, which are created as SVG.

## [Upgrading from 3.0](UPGRADE.md)

## Security

If you discover any security related issues, please email darkghosthunter@gmail.com instead of using the issue tracker.

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.

Laravel is a Trademark of Taylor Otwell. Copyright © 2011-2021 Laravel LLC.
