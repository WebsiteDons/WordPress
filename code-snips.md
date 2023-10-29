# WordPress Stuff
Anything relative to WP

## Check if plugin is installed and activated
```php
function plugExist($plugname) {
	if( file_exists(WP_PLUGIN_DIR.'/'.$plugname) && is_plugin_active($plugname.'/'.$plugname.'.php') ) {
		return true;
	}else{
		return;
	}
}
```
## Identify post type on any screen
The function `get_post_type()` is not always available so this process will cycle various methods depending on screen in view

```php
// find post type where ever
function _post_type() 
{
	global $post, $parent_file, $typenow, $current_screen, $pagenow;

	$post_type = NULL;

	if($post && (property_exists($post, 'post_type') || method_exists($post, 'post_type')))
	$post_type = $post->post_type;

	if(empty($post_type) && !empty($current_screen) && (property_exists($current_screen, 'post_type') || method_exists($current_screen, 'post_type')) && !empty($current_screen->post_type))
	$post_type = $current_screen->post_type;

	if(empty($post_type) && !empty($typenow))
	$post_type = $typenow;

	if(empty($post_type) && function_exists('get_current_screen'))
	$post_type = get_current_screen();

	if(empty($post_type) && isset($_REQUEST['post']) && !empty($_REQUEST['post']) && function_exists('get_post_type') && $get_post_type = get_post_type((int)$_REQUEST['post']))
	$post_type = $get_post_type;

	if(empty($post_type) && isset($_REQUEST['post_type']) && !empty($_REQUEST['post_type']))
	$post_type = sanitize_key($_REQUEST['post_type']);

	if(empty($post_type) && 'edit.php' == $pagenow)
	$post_type = 'post';

	return (is_string($post_type) ? $post_type : '');
}

## Determine if screen is new post or edit
Solely to make a quick use condition 

```php
function is_postMode() {
	global $pagenow;
	
	$postmode = false;
	if( 'post.php' === $pagenow | 'post-new.php' === $pagenow )
		$postmode = true;
	
	return $postmode;
}
```
