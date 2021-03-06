Yii Rest Extension
===================

Extension organize REST API on Yii app 

[Weavora's](http://weavora.com) Git Repo - [https://github.com/weavora/wrest](https://github.com/weavora/wrest)

**Features**:

* Fast configuration
* Provide simple REST CRUD interface (get, list, update, create, delete actions)
* CRUD actions integrate with Model scenarios

Configuration
-----

1) Download and extract source into protected/extensions/ folder.

2) There are config settings for import section below:

```php
<?php
// main.php
return array(
	...
	'import' => array(
		...
		'ext.wrest.*',
	),
	...
);
```

3) Add REST routes

```php
<?php
// main.php
...
'urlManager'=>array(
	'urlFormat'=>'path',
	'rules'=>array(
		...
		//rest url patterns
		array('api/<model>/delete', 'pattern'=>'api/<model:\w+>/<_id:\d+>', 'verb'=>'DELETE'),
		array('api/<model>/update', 'pattern'=>'api/<model:\w+>/<_id:\d+>', 'verb'=>'PUT'),
		array('api/<model>/list', 'pattern'=>'api/<model:\w+>', 'verb'=>'GET'),
		array('api/<model>/get', 'pattern'=>'api/<model:\w+>/<_id:\d+>', 'verb'=>'GET'),
		array('api/<model>/create', 'pattern'=>'api/<model:\w+>', 'verb'=>'POST'),
		...
	),
),
...
```

4) All your models must use WRestModelBehavior:

```php
<?php
// /models/User.php
class User extends CActiveRecord{
	...
	public function behaviors()
	{
		return array(
			...
			'RestModelBehavior' => array(
				'class' => 'WRestModelBehavior',
			)
			...
		);
	}
	...
}
```

5) Create 'api' folder in /protected/controllers/. Here will be determined all your REST API controllers.

6) Extend all your REST controlles from WRestController instead of Controller.

Usage
-----

1) Create model, determine relations, behavior etc.

```php
<?php
// /models/User.php
class User extends CActiveRecord
{

	public function tableName()
	{
		return "user";
	}

	public static function model($className = __CLASS__)
	{
		return parent::model($className);
	}

	public function behaviors()
	{
		return array(
			'RestModelBehavior' => array(
				'class' => 'WRestModelBehavior',
			)
		);
	}

	public function rules(){
		return array(
			array('username, password', 'required'),
			array('email', 'safe'),
		);
	}

}
```

2) Create controller in /protected/controllers/api folder, extend it's from WRestController and define allow rest actions

```php
<?php 
// api/UserController.php

class UserController extends WRestController{

	protected $_modelName = "user"; //model to be used as resource


	public function actions() //determine which of the standard actions will support the controller
	{
		return array(
			'list' => array( //use for get list of objects
				'class' => 'WRestListAction',
				'filterBy' => array( //this param user in `where` expression when forming an db query
					'account_id' => 'account_id', // 'name_in_table' => 'request_param_name'
				),
				'limit' => 'limit', //request parameter name, which will contain limit of object
				'page' => 'page', //request parameter name, which will contain requested page num
				'order' => 'order', //request parameter name, which will contain ordering for sort
			),
			'delete' => 'WRestDeleteAction',
			'get' => 'WRestGetAction',
			'create' => 'WRestCreateAction', //provide 'scenario' param
			'update' => array(
				'class' => 'WRestUpdateAction',
				'scenario' => 'update', //as well as in WRestCreateAction optional param
		);
	}
}
```

REST API call samples
=======================

If you use RestClient class, then you can call api next way:

```php
<?php
// apiCallSample.php

require 'RestClient.php';
$client = new RestClient('http://api_url/api/', '', '');

$response = $client->get('user'); //return the list of objects {{id:1, username:'Jack'}, {id:2, username:'Nick'}}
$response = $client->get('user/1'); //return single object with requested id, for example {id:1, username:'Jack'}
$response = $client->update('user/1', array('username' => 'new user name')); //update user data
$response = $client->delete('user/1'); //delete user with requested id
$response = $client->post('user', array('username'=>'name', 'password' => 'pass', 'email' => 'email@email.com'));//create new user

```



Custom action sample
--------------------

```php
<?php
// api/UserController.php

class UserController extends WRestController{

	protected $_modelName = "user"; //model to be used as resource
	
	...
	
	public function actionFriends($id)
	{
		$user = User::model()->findByPk($id);
		$friends = $user->findFriends();
		
		if(empty($user))
			$this->sendResponse(404);
			
		$users = array();
		foreach($friends as $friend){
			$users = $friend->getAllAttributes();
		}
		$this->sendResponse(200, $users);
	}
	
	...
	
}
```

Use with Access layer filter
-----------------

For provide access layer for REST actions use [wacf](https://github.com/weavora/yii-wacf)

```php
<?php
// api/UserController.php

class UserController extends WRestController
{

	protected $modelName = "user";

	public function filters()
	{
		return array(
			'accessControl',
		);
	}

	public function accessRules()
	{
		return array(
			array(
				'allow',
				'actions' => array('get', 'list', 'create'),
				'roles' => array('member'),
			),
			array(
				'allow',
				'roles' => array('member'),
				'actions' => array('delete', 'update'),
				'resource' => array(
					'model' => 'User',
					'params' => 'id',
					'ownerField' => 'account_id',
				),
			),
			array(
				'deny',
			),
		);
	}

	public function actions()
	{
		return array(
			'list' => array(
				'class' => 'WRestListAction',
				'filterBy' => array(
					'account_id' => 'account_id',
				),
				'limit' => 'limit',
			),
			'delete' => 'WRestDeleteAction',
			'get' => 'WRestGetAction',
			'create' => 'WRestCreateAction',
			'update' => 'WRestUpdateAction',
		);
	}
}
```