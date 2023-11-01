# Add custom tabs to product data block
```php
// add tabs in admin when creating an item
add_filter('woocommerce_product_data_tabs', function($tabs) 
{
	// remove woocommerce marketplace ad
	unset($tabs['marketplace-suggestions']);
	remove_action('woocommerce_product_data_panels', ['WC_Marketplace_Suggestions','product_data_panels']);
	
	// add custom tab
	$tabs['mycustom_tab'] = [
	'label' => 'My Tab',
	'target' => 'my_data_block',
	'priority' => 100,
	'class' => ['my-custom-tab']
	];
	// custom content panel
	add_action('woocommerce_product_data_panels', function() {
		echo '<div id="my_data_block" class="panel woocommerce_options_panel">yeah boii</div>';
	});
	
	return $tabs;
},100);
```
