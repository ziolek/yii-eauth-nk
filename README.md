Yii EAuth extension
===================

EAuth extension allows to authenticate users with accounts on other websites.
Supported protocols: OpenID, OAuth 1.0 and OAuth 2.0.

EAuth is a extension for provide a unified (does not depend on the selected service) method to authenticate the user. So, the extension itself does not perform login, does not register the user and does not bind the user accounts from different providers.


### Why own extension and not a third-party service?
The implementation of the authorization on your own server has several advantages:

* Full control over the process: what will be written in the authorization window, what data we get, etc.
* Ability to change the appearance of the widget.
* When logging via OAuth is possible to invoke methods on API.
* Fewer dependencies on third-party services - more reliable application.


### The extension allows you to:

* Ignore the nuances of authorization through the different types of services, use the class based adapters for each service.
* Get a unique user ID that can be used to register user in your application.
* Extend the standard authorization classes to obtain additional data about the user.
* Work with the API of social networks by extending the authorization classes.
* Set up a list of supported services, customize the appearance of the widget, use the popup window without closing your application.
	

### Расширение содержит:

* The component that contains utility functions.
* A widget that displays a list of services in the form of icons and allowing authorization in the popup window.
* Base classes to create your own services.
* Ready for authenticate via Google, Twitter, Facebook and other providers.


### Supported providers "out of the box":

* OpenID: Google, Yandex(ru)
* OAuth: Twitter
* OAuth 2.0: Google, Facebook, VKontake(ru), Mail.ru(ru), Moi Krug(ru), Odnoklassniki(ru)


### Resources

* [Yii EAuth](https://github.com/Nodge/yii-eauth)
* [Yii Framework](http://yiiframework.com/)
* [OpenID](http://openid.net/)
* [OAuth](http://oauth.net/)
* [OAuth 2.0](http://oauth.net/2/)
* [loid extension](http://www.yiiframework.com/extension/loid)
* [EOAuth extension](http://www.yiiframework.com/extension/eoauth)


### Requirements

* Yii 1.1 or above
* PHP curl extension
* [loid extension](http://www.yiiframework.com/extension/loid)
* [EOAuth extension](http://www.yiiframework.com/extension/eoauth)


## Installation

* Install loid and EOAuth extensions
* Extract the release file under `protected/extensions`
* In your `protected/config/main.php`, add the following:

```php
<?php
...
	'import'=>array(
		'ext.eoauth.*',
		'ext.eoauth.lib.*',
		'ext.lightopenid.*',
		'ext.eauth.services.*',
	),
...
	'components'=>array(
		'loid' => array(
			'class' => 'ext.lightopenid.loid',
		),
		'eauth' => array(
			'class' => 'ext.eauth.EAuth',
			'popup' => true, // Use the popup window instead of redirecting.
			'services' => array( // You can change the providers and their classes.
				'google' => array(
					'class' => 'GoogleOpenIDService',
				),
				'yandex' => array(
					'class' => 'YandexOpenIDService',
				),
				'twitter' => array(
					// register your app here: https://dev.twitter.com/apps/new
					'class' => 'TwitterOAuthService',
					'key' => '...',
					'secret' => '...',
				),
				'google_oauth' => array(
					// register your app here: https://code.google.com/apis/console/
					'class' => 'GoogleOAuthService',
					'client_id' => '...',
					'client_secret' => '...',
					'title' => 'Google (OAuth)',
				),
				'facebook' => array(
					// register your app here: https://developers.facebook.com/apps/
					'class' => 'FacebookOAuthService',
					'client_id' => '...',
					'client_secret' => '...',
				),
				'vkontakte' => array(
					// register your app here: http://vkontakte.ru/editapp?act=create&site=1
					'class' => 'VKontakteOAuthService',
					'client_id' => '...',
					'client_secret' => '...',
				),
				'mailru' => array(
					// register your app here: http://api.mail.ru/sites/my/add
					'class' => 'MailruOAuthService',
					'client_id' => '...',
					'client_secret' => '...',
				),
				'moikrug' => array(
					// register your app here: https://oauth.yandex.ru/client/my
					'class' => 'MoikrugOAuthService',
					'client_id' => '...',
					'client_secret' => '...',
				),
				'odnoklassniki' => array(
					// register your app here: http://www.odnoklassniki.ru/dk?st.cmd=appsInfoMyDevList&st._aid=Apps_Info_MyDev
					'class' => 'OdnoklassnikiOAuthService',
					'client_id' => '...',
					'client_public' => '...',
					'client_secret' => '...',
					'title' => 'Odnokl.',
				),
			),
		),
		...
	),
...
```


## Usage

#### The user identity

```php
<?php

class ServiceUserIdentity extends UserIdentity {
	const ERROR_NOT_AUTHENTICATED = 3;

	/**
	 * @var EAuthServiceBase the authorization service instance.
	 */
	protected $service;
	
	/**
	 * Constructor.
	 * @param EAuthServiceBase $service the authorization service instance.
	 */
	public function __construct($service) {
		$this->service = $service;
	}
	
	/**
	 * Authenticates a user based on {@link username}.
	 * This method is required by {@link IUserIdentity}.
	 * @return boolean whether authentication succeeds.
	 */
	public function authenticate() {		
		if ($this->service->isAuthenticated) {
			$this->username = $this->service->getAttribute('name');
			$this->setState('id', $this->service->id);
			$this->setState('name', $this->username);
			$this->setState('service', $this->service->serviceName);
			$this->errorCode = self::ERROR_NONE;		
		}
		else {
			$this->errorCode = self::ERROR_NOT_AUTHENTICATED;
		}
		return !$this->errorCode;
	}
}
```

#### The action

```php
<?php
...
	public function actionLogin() {
		$service = Yii::app()->request->getQuery('service');
		if (isset($service)) {
			$authIdentity = Yii::app()->eauth->getIdentity($service);
			$authIdentity->redirectUrl = Yii::app()->user->returnUrl;
			$authIdentity->cancelUrl = $this->createAbsoluteUrl('site/login');
			
			if ($authIdentity->authenticate()) {
				$identity = new ServiceUserIdentity($authIdentity);
				
				// successful authentication
				if ($identity->authenticate()) {
					Yii::app()->user->login($identity);
					
					// special redirect with closing popup window
					$authIdentity->redirect();
				}
				else {
					// close popup window and redirect to cancelUrl
					$authIdentity->cancel();
				}
			}
			
			// Something went wrong, redirect to login page
			$this->redirect(array('site/login'));
		}
		
		// default action code...
	}
```

#### The view

```php
<h2>Do you already have an account on one of these sites? Click the logo to log in with it here:</h2>
<?php 
	$this->widget('ext.eauth.EAuthWidget', array('action' => 'site/login'));
?>
```


## License

Some time ago I developed this extension for [LiStick.ru](http://listick.ru) and I still support the extension.

The extension was released under the [New BSD License](http://www.opensource.org/licenses/bsd-license.php), so you'll find the latest version on [GitHub](https://github.com/Nodge/yii-eauth).
