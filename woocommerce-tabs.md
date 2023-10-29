## WooCommerce Product Tabs ShortCode

See custom function at https://github.com/WebsiteDons/WordPress/blob/main/code-snips.md

```php
// single shop item tabs
add_shortcode('shoptab', function($attrib,$content) 
{
	if( !plugExist('woocommerce') || 'product' !== _post_type() || !is_product() )
		return;
	
	add_filter('woocommerce_product_tabs', function($tabs) use($attrib,$content) 
	{
		if( !isset($attrib['id']) )
			return $tabs;
		
		$icon= $icon2='';
		if( isset($attrib['icon']) ) 
		{
			$tabicon = $attrib['icon'];
			// define icon horizontal position of title
			if( strstr($tabicon,',') ) {
				list($ic,$p) = explode(',',$tabicon);
				$icon2 = ' <span class="tabicon '.$ic.'"></span>';
			}else{
				$icon = '<span class="tabicon '.$tabicon.'"></span> ';
			}
		}
		
		$tabs['shoptabs_'.$attrib['id']] = [
		'title' 	=> (!empty($attrib['title']) ? $icon.'<span>'.$attrib['title'].'</span>'.$icon2 : 'Tab'),
		'priority' 	=> (isset($attrib['position']) ? (int)$attrib['position'] : 100),
		'callback' 	=> (function() use($content) {
			echo do_shortcode($content);
			})
		];

		return $tabs;
	});
});
```

## ShortCode Method
**This only display in product page view**
```txt
[shoptab title="More Detail" position="1" id="more" icon="fa fa-play,1"]
this is the latest thing added
[/shoptab]

[shoptab title="It is her" id="tab2" icon="bi bi-alarm-fill"]
see what it is [videoplayer]
[/shoptab]
```
### Display result
![woocommerce-custom-tabs](https://github.com/WebsiteDons/WordPress/assets/42153624/b5cb90e3-a2b0-4ccf-a6de-b169349a14f0)
