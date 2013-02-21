# How to convert a phpBB 3.0 MOD into a phpBB 3.1 Extension


This guide should give a quick overview on the tasks for MOD-Authors in order to make a 3.0 MOD running as a 3.1 Ext, taking NV Newspage as an example.

## Extension Structure

The most obivious change should be the location, where the MODs/Extensions are stored in 3.1. In phpBB 3.0 you just put all the files into the core's root folder. As for 3.1 we created a special directory for Extensions. It's called **ext/**.

### Directory

Each extension has it's own directory. However you can (and should) also use an additional vendor directory (with your author name or author-group name). So in case of my newspage the files will be in

> phpBB/ext/nickvergessen/newspage/

You should not need any files to be located outside of that directory. No matter which files, may it be styles, language or ACP module files. All of them will be moved into your extension's directory.

			new directory			| current directory
			------------------------+----------------
	.../newspage/					|
			acp/					| phpBB/includes/acp/
									| phpBB/includes/acp/info/
			adm/style/				| phpBB/adm/style/
			config/					|	---
			controller/				|	---
			language/				| phpBB/language/
			migration/				|	---
			styles/					| phpBB/styles/

I already listed some additional new directories here which will be explained later.

### Front-facing files, routes and services

While in 3.0 you just created a new file in the root directory of phpBB, you might want to use the new controller system of 3.1 in future. Your links change from something like `phpBB/newspage.php` to `phpBB/app.php?controller=newspage` in first place, but with a little htaccess rule this can be rewritten to `phpBB/newspage`

In order to link a specific routing rule to your extension, you need to define the route in your extension's **config/routing.yml**

For the easy start of the newspage, 2 rules are enough. The first rule is for the basic page currently `newspage.php`, the second one is for the pagination, like `newspage.php?start=5`. The first rule sets a default page (1), while the second rule requires a second part of the url to be an integer.


	newspage_base_controller:
	    pattern: /newspage/
	    defaults: { _controller: nickvergessen.newspage.controller:base, page: 1 }

	newspage_page_controller:
	    pattern: /newspage/{page}/
	    defaults: { _controller: nickvergessen.newspage.controller:base }
	    requirements:
	        page:  \d+

The string we define for `_controller` defines a service (`nickvergessen.newspage.controller`) and a method (`base`) of the class which is then called. Services are defined in your extensions **config/services.yml**. Services are instances of classes. Services are used, so there is only one instance of the class which is used all the time. You can also define the arguments for the constructor of your class. The example definition of the newspage controller service would be something similar to:

	services:
	    nickvergessen.newspage.controller:
	        class: phpbb_ext_nickvergessen_newspage_controller_main
	        arguments:
	            - @auth
	            - @cache
	            - @config
	            - @dbal.conn
	            - @request
	            - @template
	            - @user
	            - @controller.helper
	            - %core.root_path%
	            - %core.php_ext%

Any service that is previously defined in your file, or in the file of the phpBB core `phpBB/config/services.yml`, can also be used as an argument, aswell as some predefined string (like `core.root_path` here).

**NOTE:** The classes from phpBB/ext/ are automatically loaded by their names, whereby underscored ( _ ) can represent directories. In this case the class `phpbb_ext_nickvergessen_newspage_controller_main` would be located in `phpBB/ext/nickvergessen/newspage/controller/main.php`

For more explanations about [Routing](http://symfony.com/doc/2.1/book/routing.html) and  [Services](http://symfony.com/doc/2.1/book/service_container.html) see the Symfony 2.1 Documentation.

In this example my **controller/main.php** would look like the following:

	<?php
	
	/**
	*
	* @package NV Newspage Extension
	* @copyright (c) 2013 nickvergessen
	* @license http://opensource.org/licenses/gpl-2.0.php GNU General Public License v2
	*
	*/
	
	class phpbb_ext_nickvergessen_newspage_controller_main
	{
		/**
		* Constructor
		* NOTE: The parameters of this method must match in order and type with
		* the dependencies defined in the services.yml file for this service.
		*
		* @param phpbb_auth		$auth		Auth object
		* @param phpbb_cache_service	$cache		Cache object
		* @param phpbb_config	$config		Config object
		* @param phpbb_db_driver	$db		Database object
		* @param phpbb_request	$request	Request object
		* @param phpbb_template	$template	Template object
		* @param phpbb_user		$user		User object
		* @param phpbb_controller_helper		$helper		Controller helper object
		* @param string			$root_path	phpBB root path
		* @param string			$php_ext	phpEx
		*/
		public function __construct(phpbb_auth $auth, phpbb_cache_service $cache, phpbb_config $config, phpbb_db_driver $db, phpbb_request $request, phpbb_template $template, phpbb_user $user, phpbb_controller_helper $helper, $root_path, $php_ext)
		{
			$this->auth = $auth;
			$this->cache = $cache;
			$this->config = $config;
			$this->db = $db;
			$this->request = $request;
			$this->template = $template;
			$this->user = $user;
			$this->helper = $helper;
			$this->root_path = $root_path;
			$this->php_ext = $php_ext;
	
			if (!class_exists('bbcode'))
			{
				include($this->root_path . 'includes/bbcode.' . $this->php_ext);
			}
			if (!function_exists('get_user_rank'))
			{
				include($this->root_path . 'includes/functions_display.' . $this->php_ext);
			}
		}
	
		/**
		* Base controller to be accessed with the URL /newspage/{page}
		* (where {page} is the placeholder for a value)
		*
		* @param int	$page	Page number taken from the URL
		* @return Symfony\Component\HttpFoundation\Response A Symfony Response object
		*/
		public function base($page = 1)
		{
			/*
			* Do some magic here,
			* load your data and send it to the template.
			*/			

			/*
			* The render method takes up to three other arguments
			* @param	string		Name of the template file to display
			*						Template files are searched for two places:
			*						- phpBB/styles/<style_name>/template/
			*						- phpBB/ext/<all_active_extensions>/styles/<style_name>/template/
			* @param	string		Page title
			* @param	int			Status code of the page (200 - OK [ default ], 403 - Unauthorized, 404 - Page not found)
			*/
			return $this->helper->render('newspage_body.html');
		}
	}

You can also have multiple different methods in one controller aswell as having multiple controllers, in order to organize your code a bit better.

If we now add the entry for our extension into the phpbb_ext table, and go to `example.tld/app.php?controller=newspage/` you can see your template file. **Congratulations!** You just finished the "Hello World" example for phpBB Extensions. ;)