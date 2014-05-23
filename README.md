Rust Framework
======

I created this very basic, bare bones PHP RESTFul Framework that minimizes dependencies while providing variable validation and rapid adaptation via inversion of control. 

Rationale/Background
------
I was motivated to come up with something that was simple yet powerful. I also wanted to keep the dependencies to a bare minimum and be really flexible for developers, but work within the limits/features of PHP. I also wanted something that was easy to figure out and easy to determine where the logic was. One big problem I have with my legacy code is figuring out where to start because our entry points aren't clearly defined given different implementations. What I found in Billy's implementation was the notion of a route that defined everything (at least that's where I've taken it.) The route defines the URI, HTTP Method, class and method to execute as well as the class to use for standard out and standard error. I also wanted to reduce inheritance and limit the number of files you have to open to figure out what's going on. I think there has been a bit of an abuse of inheritance in the current php rest framework so much of our code uses. Inheritance for the sake of inheritance is a bad idea. The framework lets you use inheritance or composition, but it's actually less characters to type if you use composition instead of inheritance. 

How it works
------
Typically you let your web server locate files off of the doc root. At the Times, we tend to route requests in Apache using Alias and regular patterns e.g. AliasMatch /svc/xyz /some/php/entry/point

What you do after that point varies and the framework I built address what to do after you have mapped the request to the entry point. In this framework you have to do a few things.

  - Install the framework
  - Define your route
  - Create your application logic
  - Pass your routes to the controller

With the barebones rest framework, that's really all you need to do. 

Why use it?
------

I found a few great benefits developing an API with this framework.

  - Quick and easy to make changes. If you need to change the URL, add parameters to the url or to the post body its really quick and easy since it's just a configuration change.
  - Zero variable validation in my code. It really cleans up your code. You go from having a lot of error checking in each method to only checking to see if stuff came in e.g. paging values don't have to bet passed, but if they do, you don't get away from checking for is_set or empty.
  - Quickly able to add new standard out/standard error handlers without breaking everything else. When I added "help" I was able to add a straight json output in seconds.
  - Self documenting. If you map in help, you can spit out the key API information as JSON. I'm currently working with Eric Schorr to make it look pretty.

Using the Google CSE API as an example
------
Again, the framework requires very little to use. Here's the most basic example.

The web server entry point for the API is index.php.  Map this to a URL path on your server.
  * RewriteRule ^/svc/cse /opt/nyt/www/api/external/svc/cse/index.php [L] (Apache)
  * server { location /svc/cse { fastcgi_param  SCRIPT_FILENAME  $document_root/index.php; } } (Nginx)

The entry point maps the routes (URL paths) to the class/method to be invoked, and the corresponding response objects.  It passes those route definitions to the controller, which invokes the service.

  * Google CSE API index.php: https://svn.prvt.nytimes.com/svn/app/google-cse-api/trunk/src/www/api/external/svc/cse/index.php
  * Google CSE API Search.class.php: https://svn.prvt.nytimes.com/svn/app/google-cse-api/trunk/src/lib/websafe/php/NYTD/GoogleCSESearch/API/Search.class.php (invoked method accepts the decoded GET/POST params from the controller)

```
<?php
/** index.php
 *
 * @author James Boehmer <james.boehmer@nytimes.com>
 * @version $Id: index.php 66242 2012-08-09 21:55:08Z james.boehmer $
 */

nytd_require('NYTD/Rust/Service/Controller.php');
nytd_require('NYTD/GoogleCSESearch/API/Search.class.php');
nytd_require('NYTD/GoogleCSESearch/API/Response.class.php');

$routes = array(
        array('rule'   => ';^/svc/cse/v2/sitesearch.jsonp$;',
            'params'  => array(),
            'action'  => 'GET',
            'class'   => 'NYTD_GoogleCSESearch_API_Search',
            'method'  => 'cmdGoogleCSESearch',
            'name'    => 'Site Search',
            'docs'    => 'Invoke a Google site search',
            'std_out' => 'NYTD_GoogleCSESearch_API_JSONPResponse',
            'std_err' => 'NYTD_GoogleCSESearch_API_ErrorResponse',
            'pcheck' => array('callback' => '/(.+)/'),
        ),
        array('rule'   => ';^/svc/cse/v2/sitesearch.json$;',
            'params'  => array(),
            'action'  => 'GET',
            'class'   => 'NYTD_GoogleCSESearch_API_Search',
            'method'  => 'cmdGoogleCSESearch',
            'name'    => 'Site Search',
            'docs'    => 'Invoke a Google site search',
            'std_out' => 'NYTD_GoogleCSESearch_API_JSONResponse',
            'std_err' => 'NYTD_GoogleCSESearch_API_ErrorResponse',
            'pcheck' => array(),
        ),
        
);
$serviceController = new NYTD_Rust_Service_Controller();
$result = $serviceController->run($routes);
```

A quick restart/reload of Apache and you can see it working.
```
  curl http://localhost/svc/cse/help.json
```
I've been trying to keep it simple, and I feel like that goal is accomplished. As you can see above, you don't need to require but a few files. If you wanted to go even more slim, you can take out the Success and Error response requires, remove the std_out and std_err from the route and handle the results by hand. I went with the classes because I generally write RESTFul APIs that have specific output expectations that don't change from method to method. 

Routes
------  
As of this writing, a route consists of the following attributes:

  * rule
  * method
  * params
  * class
  * action
  * pcheck
  * title
  * docs
  * std_out
  * std_err
  * filters

**rule**: The rule is a regular pattern expression e.g. 
```
   ;^/svc/asset/user/([0-9]{1,15})/license/count.json$;
```
**action**: what HTTP method. checks $_SERVER['REQUEST_METHOD']

**params**: this is where you can name the elements in the url. e.g. ([0-9]{1,15}) is regi_id or age

**class**: this is the class file the code should create a new instance of. 

**method**: this is the method of the class to call. When the method is called, the controller will pass in the validated data as a array/hash

**title**: this is for the help feature, lets you name the service

**docs**: this is where you give a brief overview of service the class/method provides

**pcheck**: this is where you put variable validation.

**std_out**: you can define a class that handles good results

**std_err**: you can define a class that handles bad results

**filters**: you can define one or more classes to filter for basic input like valid NYT-S cookie. The controller will pass in the address of the parameters to allow filters to add additional parameters. Filters return TRUE if all is well or should pass back array(HTTP_RESPONSE_CODE=>message)


pcheck
---
This requires a little documentation. Let's take a look at an example:

```
array('rule'   => ';^/svc/asset/user/([0-9]{1,15})/license/type/([a-z]{1,40})/size/([0-9]{1,10}).json$;',
                    'params' => array('script_path','id','asset_type','views'),
                    'action' => 'POST',
                    'class'  => 'NYTD_Rust_AssetAccess_License',
                    'method' => 'restCreateUserLicense',
                    'std_out' => 'NYTD_Rust_MetaJson_Response',
                    'std_err' => 'NYTD_Rust_MetaJson_ErrorResponse',
                    'pcheck' => array('subscription_id'=>'/([0-9]{1,15})/',
                                      'asset_type'     =>'/([a-z]{1,15})/',
                                      '*views'         =>'/([0-9]{1,10})/',
                                      '*does_expire'   =>'/(true|false)/',
                                      '*offer_chain_id'=>'/([0-9]{1,15})/',
                                      '*expires_on'    =>'/([0-9]{4})-([0-9]{2})-([0-9]{2}) ([0-9]{2}):([0-9]{2}):([0-9]{2})/',
                                      '*expires'       =>'/([0-9]{4})-([0-9]{2})-([0-9]{2}) ([0-9]{2}):([0-9]{2}):([0-9]{2})/',
                                      '*starts'        =>'/([0-9]{4})-([0-9]{2})-([0-9]{2}) ([0-9]{2}):([0-9]{2}):([0-9]{2})/',
                                      '*#subscription_meta_data' =>array('*CIS_PRD_ID'=>'/([A-Z0-9]{0,2})/'),
                                      '*#offer_chain_meta_data'  =>array('*PRD_ID'=>'/^([0-9]{0,15})$/'),
                                      '*!ignore'       =>''
                                      )
                    ),
              
              );
```

pcheck should always be a hash (an array with named keys), even if it's empty. 
pcheck is deinfed as "variable name"=>"regex" OR "variable name"=>hash.
Rather than create structured json, I opted to use special charachters to designate rule features.
Here are the rule features.

  - If you define a pcheck rule, then the hash should have a key/value pair that will be validated with the RE.
  - * If the name starts with *, the variable is optional. If it's present, it will be validated against the RE.
  - # If the name start #, the variable is expected to be another hash.
  - ! If the name starts with !, then the variable is ignored. You can do this to document the variable but not apply any validation
  - @ If the name starts with @, then the variable is repeated e.g. a simple array. array(3,2,6,9)

The code will look in the above order, so order is important. If you want to define an optional hash,
you would the following:

```
"*#dimensions"=>array('height'=>[0-9]{1-3},'width'=>[0-9]{1-3})
```

std_out/std_err
---
I decided to allow you to define how to handle the output, good or bad, with these parameters. The contract I assume is array(200=>any) or array(5xx=>any) but since you can define your own standard out/standard error you can still use the framework and return whatever best suits your application. The interface for std_out and std_err is to implement a constructor

```
public function __construct($code,$data)
```

Based on the std_err/std_out interface and application logic method returns, the framework has the following lines of code:

```
$hclass  = $route['class'];
$method  = $route['method'];
$handler = new $hclass;

if ($method == 'help' && $hclass == 'NYTD_Rust_Service_Controller') {
    /*
     * a special case to spit out the route data sharing what services the routes provide
     */
    $result  = $handler->$method($routes);
} else {
    $result  = $handler->$method($this->params);
}

/*
 * If std_out and std_err are defined, handle the results
 */
if (!empty($route['std_out']) && !empty($route['std_err']) ) {
    if (empty($result[NYTD_Rust_HTTP_ResponseCodes::GOOD])) {
        $err = $route['std_err'];
        foreach ($result as $code=>$msg)
            $res = new $err($code,$msg);
    } else {
        $out = $route['std_out'];
        $res = new $out($result[NYTD_Rust_HTTP_ResponseCodes::GOOD]);
    }
    return;
}
return $result;
```

Application Logic
------
In order for inversion of control to work, the most important piece of information to decide on was how to pass data between the framework and the code a user would write. Rather than get overly complicated, I decided to go with an array. So, if you are going to use the framework and you want to use built in standard out/standard error then your methods should **ALWAYS** return an array using HTTP Respons codes. A good response would be array(200=>any) or array(500=>any) This means that you can return any type of data in the array, but the contract is to pass the success/error in the hash when your method.

To illustrate what we expect on success or failure:

```
public function stubSuccess($params) {
  return array(200=>array());
}

public function stubFailure($params) {
  return array(500=>"That call failed on purpose");
}
```

What if I don't like that contract, can I still use the framework? Yes. Do **NOT** specify std_out and std_in, and then simply inspect the return value from the $run method. The framework will create an instance of your class, execute the method and pass back the results. 

The Controller
------
```
src/lib/backend/php/NYTD/Rust/Service/Controller.php
```

The controller file has the logic to grab and process input data and then map requests to routes. To use the framework, you have to either extend the Controller or create an instance and execute the run method. e.g.

```
    /**
     * The run function normally expects no parameters, adding url and params
     * to support test frameworks like simple test.
     *
     * @param $path   - support unit tests
     * @param $params - support unit tests
     */
    public function run($path=null, $params=null) {
        $r = new NYTD_Rust_Service_Controller();
        $result = $r->run($this->routes,$path,$params);
    }
```

When executed in that way, it is assumed that the route defines the std-out or the std-err. If they are not defined you would have to look at the responses yourself. 

**NOTE** One major assumption I worked off of was that the methods executed would always return back an array with a numeric index where the index reflects success or failure using http response codes. e.g.

```
if (!empty($result['200'])) {
  // it worked
} else {
  // best to foreach
  foreach ($result as $code=>$data) {
    //.. handle error
  }
}
```

If you have some legacy code that doesn't pass anything back or does something different, that's o.k. too.

Feel free to look under the hood. Since I wrote this for me, I took the liberty to add logic when processing input. For instance, I pass all input to the Controller::decodeInput method. In there I look at the data and if it starts and ends with {} or [] then I assume the data to be JSON and encode the JSON. In addition, since we have agreed that all RESTFul JSON will be {meta:{},data:{}} and I don't typically need the meta I added a feature to strip data. Say you pass in {meta:{},data:{reg_id:12345}} the framework will pass along only {regi_id:12345} to my application logic.

```
public function __construct($strip=TRUE,$params=null,$action='GET') {
```

So, if you want the meta/data structure in your data, or if you like to pass in using the parameter data, then you should construct an instance of the controller with:

```
$r = new NYTD_Rust_Service_Controller(false);
```

The validator 
------
```
src/lib/backend/php/NYTD/Rust/Hash/Validator.php
```

The validator is the code that checks the parameters against the regular pattern expressions. I separated out the variable validation logic if you want to use the validates on it's own.