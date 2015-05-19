# yii-dbeanstalkd
Distributed wrapper for yii for davidpersson's beanstalkd client

This project differs from other yii extensions in that it provides a very trivial method of 
managing multiple, distributed beanstalkd servers.  Other projects, or classes, differ in that they build
upon existing implementations but do not offer any method of failover or recovery from a server failure.

Since a single server going down would cause complete failure this must, at the very least:

	* be aware of multiple servers/connections
	* be able to choose between them based upon a simple weighted algorithm
	* be able to use a different server if one is unavailable

In the event of a server failure, this class makes no attempt to ensure enqueued items
are not lost since.  This will allow the user to enqueue in another server and keep going.
 
This class extends upon great work from the following individuals or projects:

	* davidpersson's beanstalk project: <https://github.com/davidpersson/beanstalk>
	* Yiinstalk, a yii extension for beanstalkd: <https://github.com/shiki/Yiinstalk>
	* yii2-beanstalk, a yii2 extension for beanstalkd: <https://github.com/udokmeci/yii2-beanstalk>

## Forewarning
This is my first contribution to github and to the world.  I'm not perfect and I am already busy at work.
If this helps you in some way then I will have been able to give back for all of those times I have 
borrowed someone else's fine work on the web.  If you are able to improve this project or correct some
mis-step or outright mistake on my part I will be grateful.  Thank you, and good luck.

## Usage
This project requires:
	* davidpersson's beanstalkd client to be added to your yii project
	* modifications to config files to support these additions

### adding davidpersson's class to your project
Clone davidpersson's library and place it in the extensions folder of your yii application, 
in a folder named 'beanstalk'.

### adding this extension to your config file
Ensure that the Beanstalk.php class from this project has been placed in your components folder.

Ensure this line has been added under the aliases section of your yii config file:
``` php
'aliases'=>array(
		// davidpersson's library has been installed to extensions/beanstalk folder
		'Beanstalk'=> 'application.extensions.beanstalk.src',
	),
```

Ensure these lines have been added to the components section of your yii config file:
``` php
'components'=>array(
		'beanstalk'=>array(
			'class'=>'application.components.Beanstalk',
			'servers'=>array(
				'server1'=>array(
					'host'=>'127.0.0.1',
					'port'=>11300,
					'weight'=>50,
					// array of connections/tubes
					'connections'=>array(),
				),
				'server2'=>array(
					'host'=>'127.0.0.1',
					'port'=>11300,
					'weight'=>50,
					// array of connections/tubes
					'connections'=>array(),
				),
			),
		),
),
```

After this point one should be able to make use of beanstalk functions by doing the following:

#### beanstalkd producer example

``` php
/*
 * connect to a server (determines by weight or selects only existing server::
 */
$client = Yii::app()->beanstalk->getClient();
$client->connect();

/* 
 * alternatively, one can connect to a server by name:
 */
//$client = Yii::app()->beanstalk->getClient('server1');
//$client->connect();

$client->useTube('default');

//echo "Tube used: {$client->listTubeUsed()}.\n";
//print_r($client->stats());

$jobDetails = [
	'application'=>'beanstalk test',
	'payload'=>'performing a test on: '.date('Y-m-d H:i:s'),
];
$jobDetailString = json_encode($jobDetails);

$ret = $client->put(
	0, // priority
	0, // do not wait, put in immediately
	90, // will run within n seconds
	$jobDetailString // job body
);

echo "Added $jobDetailString to queue.\n";
echo "Return was: $ret.\n";

$client->disconnect();
```

#### beanstalkd consumer example

##### peeking
``` php
$peek = Yii::app()->beanstalk->getClient();
$peek->connect();
$peek->watch('default');

$job = $peek->reserve(5);
print_r($job);

$result = touch($job['body']);
print_r($job);
print_r($result);
$peek->disconnect();
```

##### consuming
``` php
$consumer = Yii::app()->beanstalk->getClient();
$consumer->connect();
$consumer->watch('default');

while(true)
{
	$job = $consumer->reserve();
	$result = touch($job['body']);

	if( $result )
	{
		// do something with the job request
		echo "Done...\n";
		$consumer->delete($job['id']);
	}
	else
	{
		// handle failure here
		echo "Burying...\n";
		$consumer->bury($job['id']);
	}
}

$consumer->disconnect();
```
