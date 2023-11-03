# Bulk price change

```php
<?php
/*
A process to set bulk dislcounts in WooCommerce
*/
defined('ABSPATH') || exit();

// create menu item in products navigation
add_action('admin_menu', function() {
	add_submenu_page(
	'edit.php?post_type=product', 
	'Bulk price change',
	'Bulk Discount',
	'manage_options',
	'bulk-price-change',
	(function() {
		bulk_price_form();
	})
	);
},100);

// html output
function bulk_price_form()
{
	// create category select options array
	$cats=[];
	foreach(get_terms('product_cat') as $cat) {
		$cats[] = '<option value="'.$cat->term_id.'">'.$cat->name.'</option>';
	}
	
	$confirm_table=[];
	// define temporary text file to store discounted prices as JSON array
	$temp_file = wp_upload_dir()['basedir'].'/tmp_discount_price';
	$endpoint = '';
	
	if( !empty($_POST) ) 
	{
		if( !empty($_POST['cats']) )
		{
			$items = woo_items($_POST['cats']);
			$discount_value = (int)$_POST['discount_value'];
			$discount_type = $_POST['discount_type'];
			$new_prices= [];
			
			foreach($items as $item) 
			{
				$reg_price = (int)get_post_meta($item,'_regular_price',true);
				
				if( !empty($discount_value) && 'clear_discount' !== $discount_type )
				{
					if( $discount_type == 'fixed_amount' ) {
						$new_price = ($reg_price - $discount_value);
					}else{
						$percentage = ($reg_price * $discount_value / 100);
						$new_price = ($reg_price - $percentage);
					}
					
					$new_prices[$item] = $new_price;
					$confirm_table[] = '<tr><td>'.get_the_title($item).'</td><td>'.$reg_price.'</td><td>'.$new_price.'</td></tr>';
				}
				
				if( 'clear_discount' === $discount_type ) {
					update_post_meta($item,'_price',$reg_price);
					delete_post_meta($item,'_sale_price');
					$endpoint = '&confirm=1';
				}
			}
			
			// store new prices in temporary text file for use on confirmation page
			// the file is deleted once confirmation process is submitted
			if( !empty($new_prices) )
				file_put_contents($temp_file,json_encode($new_prices));
		}
		
		// confirmation process
		if( isset($_POST['confirm_price_update']) ) 
		{
			if( !file_exists($temp_file) )
				return;
			
			$new_prices = file_get_contents($temp_file);
			$new_prices = json_decode($new_prices,true);
			if( !empty($new_prices) ) 
			{
				foreach($new_prices as $pid => $discount_price) {
					update_post_meta($pid,'_price',$discount_price);
					update_post_meta($pid,'_sale_price',$discount_price);
				}
				
				// delete temp file
				unlink($temp_file);
			}
		}
	}
	
	// create confirmation table for review before sending data
	if( isset($_GET['confirm']) ) 
	{
		?>
		<h2>Confirm price changes</h2>
		<form action="edit.php?post_type=product&page=bulk-price-change&confirm=1" method="post">
			<table class="wp-list-table widefat fixed striped table-view-list posts">
				<tr><th>Title</th><th>Regular Price</th><th>Discount Price</th></tr>
				<?php echo implode($confirm_table); ?>
			</table>
			<div class="controls">
			<?php 
			submit_button('Confirm Prices Update','primary','confirm_price_update'); 
			?>
			<p class="submit"><a href="edit.php?post_type=product&page=bulk-price-change" class="button">Cancel</a></p>
			</div>
		</form>
		<?php
	}
	else
	{
	
	?>
	<h2>Bulk Price Change</h2>
	<form action="edit.php?post_type=product&page=bulk-price-change<?php echo $endpoint; ?>" method="post">
		<div class="fieldset">
			<div class="controls">
				<label>Category</label>
				<select name="cats[]" multiple>
					<option value="">All Categories</option>
					<?php echo implode($cats); ?>
				</select>
			</div>
			<div class="controls">
				<label>Set discount value</label>
				<input type="number" name="discount_value" min="0" />
			</div>
			<div class="controls">
				<label>Discount Type</label>
				<select name="discount_type">
					<option value="fixed_percent">Percent</option>
					<option value="fixed_amount">Amount</option>
					<option value="clear_discount">Clear Discount</option>
				</select>
			</div>
			<?php 
			submit_button('Update Prices'); 
			?>
		</div>
	</form>
	<?php 
	}
	?>
	
	<style>
	.controls {display: flex; gap: 15px; margin-bottom: 15px;}
	.controls label {width: 180px !important;}
	</style>
	<?php
}

// get item ID list
function woo_items($cats)
{
	$items = get_posts([
	'post_type'=>'product',
	'tax_query'=>[[
		'taxonomy'=>'product_cat',
		'field'=>'term_id',
		'terms'=>$cats
	]],
	'showposts' => -1,
	'fields'=>'ids'
	]
	);
	
	return $items;
}

```
