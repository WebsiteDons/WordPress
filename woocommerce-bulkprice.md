# Bulk price change

### Usage
**mu-plugins option**

* Create a PHP file in `wp-content/mu-plugins` and name it relatively like **bulk-discount-price.php** If the folder _mu-plugins_ does not exist, just create it, it is a WordPress core directory that allows storage of arbitray function files. It is not created during installation.
* Copy the below code to the new file and save
* That's it

If using the theme's functions.php is preferred, only copy the coding **_after_** `defined('ABSPATH') || exit();`

```php
<?php
defined('ABSPATH') || exit('CMSEnergizer.com');

class wcDiscount 
{
	function __construct() {
		// if needed
	}
	
	// sub menu bulk discount pages
	public static function bulk_price_form()
	{
		// form request processor
		self::do_request();
		
		// create confirmation table for review before sending data
		if( isset($_GET['confirm']) ) 
		{
			?>
			<div class="wrap">
			<h2>Confirm price changes</h2>
			
			<?php do_action('get_confirm_table'); ?>
			</div>
			
			<?php
		}
		else
		{
		// set price discount form
		?>
		<div class="wrap">
		<h2>Bulk Price Change</h2>
		<p>The values set will affect all items in the selected category. The date schedule is optional. All values can be edited individually during the confirmation stage</p>
		<form action="edit.php?post_type=product&page=bulk-price-change&confirm=1" method="post">
			<div class="fieldset flex gap10 flexmid">
				<div class="control">
					<div><label>Category</label></div>
					<div>
					<select name="cats[]" multiple required>
						<?php do_action('get_shop_categories'); ?>
					</select>
					</div>
				</div>
				<div class="control">
					<div><label>Discount Type</label></div>
					<div>
					<select name="discount_type">
						<option value="fixed_percent">Percent</option>
						<option value="fixed_amount">Amount</option>
						<option value="clear_discount">Clear Discount</option>
					</select>
					</div>
				</div>
				<div class="control">
					<div><label>Set discount value</label></div>
					<div><input type="number" name="discount_value" min="0" step="0.01" /></div>
				</div>
				<div>
					<div class="flex flexcenter gap10">
						<div>
							<div><label>Start date</label></div>
							<div><input type="date" name="sale_start_date" /></div>
						</div>
						<div>
							<div><label>End date</label></div>
							<div><input type="date" name="sale_end_date" /></div>
						</div>
					</div>
				</div>
				
				<div><?php submit_button('Set Discount Prices'); ?></div>
			</div>
		</form>
		
		<?php self::get_current_discounts(); ?>
		</div>
		
		<?php 
		}
		?>
		
		<style>
		.flex-col {display: flex; gap: 5px; align-items: center;}
		.discount-clear {cursor: pointer;}
		.low-price-alert {background: #f2b8b8 !important;}
		.fieldset p.submit {padding: 0; padding-top: 5px;}
		.flex {display:flex;}
		.flexmid {align-items: center;}
		.gap5 {gap: 5px;}
		.gap10 {gap: 10px;}
		.gap15 {gap: 15px;}
		.gap20 {gap: 20px;}
		</style>
		<script>
		jQuery(function($) {
			$(".discount-clear").each(function(i,v) {
				$(this).on("click",function() {
					$(v).prev().find("input").val("");
				});
			});
		});
		</script>
		<?php
		
	}
	
	// form request process
	private static function do_request()
	{
		// create category select options array
		$cats=[];
		foreach(get_terms('product_cat') as $cat) {
			$cats[] = '<option value="'.$cat->term_id.'">'.$cat->name.'</option>';
		}
		$categories = implode($cats);
		add_action('get_shop_categories',function() use($categories) {
			echo $categories;
		});
		
		$vals=[];
		
		if( !empty($_POST) ) 
		{
			// bulk price submit
			if( !empty($_POST['cats']) )
			{
				$items = woo_items($_POST['cats']);
				$discount_value = (int)$_POST['discount_value'];
				$discount_type = $_POST['discount_type'];
				$startdate = (!empty($_POST['sale_start_date']) ? strtotime($_POST['sale_start_date']):'');
				$end_date = (!empty($_POST['sale_end_date']) ? strtotime($_POST['sale_end_date']):'');
				$setprice=false;
				$confirm_table=[];
				
				foreach($items as $item) 
				{
					$reg_price = (int)get_post_meta($item,'_regular_price',true);
					
					if( !empty($discount_value) && 'clear_discount' !== $discount_type )
					{
						$setprice = true;
						if( $discount_type == 'fixed_amount' ) {
							$new_price = ($reg_price - $discount_value);
						}else{
							$percentage = ($reg_price * $discount_value / 100);
							$new_price = ($reg_price - $percentage);
						}
						
						$confirm_table[$item] = [
						'itemtitle'=>get_the_title($item),
						'saleprice'=>$new_price,
						'regprice'=>$reg_price,
						'startdate'=>$startdate,
						'end_date'=>$end_date
						];
					}
					
					if( 'clear_discount' === $discount_type ) {
						update_post_meta($item,'_price',$reg_price);
						delete_post_meta($item,'_sale_price');
						delete_post_meta($item,'_sale_price_dates_from');
						delete_post_meta($item,'_sale_price_dates_to');
					}
				}
				
				if( 'clear_discount' === $discount_type ) {
					wp_redirect(admin_url().'edit.php?post_type=product&page=bulk-price-change');
					exit;
				}
				
				// set confirm table
				$cancel_url = admin_url().'edit.php?post_type=product&page=bulk-price-change';
				$cancel_btn = '<p class="submit"><a href="'.$cancel_url.'" class="button">Cancel</a></p>';
				add_action('date_head', function() {echo '<th>Scheduled Dates (start/end)</th>';});
				
				$vals = [
				'data'=>$confirm_table,
				'form'=>'<form action="edit.php?post_type=product&page=bulk-price-change" method="post">',
				'formclose'=>'<div class="flex flexmid gap10">'.get_submit_button('Confirm Discount Prices','primary','confirm_discount_prices').$cancel_btn.'</div></form>'
				];
			}
			
			// confirmation table html hook
			add_action('get_confirm_table', function() use($vals){
				self::table($vals);
			});
			
			// confirmation post process
			if( isset($_POST['confirm_discount_prices']) ) 
			{
				if( !empty($_POST['items']) ) 
				{
					foreach($_POST['items'] as $pid => $discount_price) 
					{
						if( empty($discount_price) )
							continue;
						
						update_post_meta($pid,'_price',$discount_price);
						update_post_meta($pid,'_sale_price',$discount_price);
						
						$date_start = $_POST[$pid.'_sale_start_date'];
						$date_end = $_POST[$pid.'_sale_end_date'];
						
						if( !empty($date_start) )
							update_post_meta($pid,'_sale_price_dates_from',strtotime($date_start));
						if( !empty($date_end) )
							update_post_meta($pid,'_sale_price_dates_to',strtotime($date_end));
					}
				}
			}
			
			
			// current prices update
			if( isset($_POST['update_discount_prices']) ) 
			{
				if( !empty($_POST['items']) ) 
				{					
					foreach($_POST['items'] as $pid => $discount_price) 
					{
						if( empty($discount_price) )
							continue;
						
						update_post_meta($pid,'_price',$discount_price);
						update_post_meta($pid,'_sale_price',$discount_price);
						
						$date_start = $_POST[$pid.'_sale_start_date'];
						$date_end = $_POST[$pid.'_sale_end_date'];
						
						if( !empty($date_start) ) {
							update_post_meta($pid,'_sale_price_dates_from',strtotime($date_start));
						}else
						if( empty($date_start) ) {
							delete_post_meta($pid,'_sale_price_dates_from');
						}
						if( !empty($date_end) ) {
							update_post_meta($pid,'_sale_price_dates_to',strtotime($date_end));
						}else
						if( empty($date_end) ) {
							delete_post_meta($pid,'_sale_price_dates_to');
						}
					}
				}
			}
		}
		
		return;
	}
	
	// get current discounts
	private static function get_current_discounts() 
	{
		if( empty(wooSaleItems()) )
			return null;
		
		$data=[];
		foreach(wooSaleItems() as $wi) 
		{
			$startdate = get_post_meta($wi->post_id,'_sale_price_dates_from',true);
			$end_date = get_post_meta($wi->post_id,'_sale_price_dates_to',true);
			
			$data[$wi->post_id] = [
			'itemtitle'=>get_the_title($wi->post_id),
			'saleprice'=>$wi->meta_value,
			'regprice'=>get_post_meta($wi->post_id,'_regular_price',true),
			'startdate'=>$startdate,
			'end_date'=>$end_date
			];
		}
		
		add_action('date_head', function() {echo '<th>Scheduled Dates (start/end)</th>';});
		
		$vals = [
		'data'=>$data,
		'title'=>'Current Discounts',
		'form'=>'<form action="edit.php?post_type=product&page=bulk-price-change&priceupdate=1" method="post">',
		'formclose'=>get_submit_button('Update Discount Prices','primary','update_discount_prices').'</form>'
		];
		
		return self::table($vals);
	}
	
	// prices table
	private static function table($data)
	{
		if( empty($data['data']) )
			return;
		
		$title = (!empty($data['title']) ? '<h3>'.$data['title'].'</h3>':'');
		$form=$formclose='';
		if( isset($data['form']) && isset($data['formclose']) ) {
			$form = $data['form'];
			$formclose = $data['formclose'];
		}
		$row = (isset($data['row']) ? $data['row'] : '');
		?>
		
		<?php echo $form; ?>
		<?php echo $title; ?>
		<table class="wp-list-table widefat fixed striped table-view-list posts">
			<tr><th>Title</th><th>Regular Price</th><th>Discount Price</th><?php do_action('date_head'); ?></tr>
			
			<?php 
			foreach($data['data'] as $id => $val) 
			{
				$startdate = (!empty($val['startdate']) ? date('Y-m-d',$val['startdate']) :'');
				$end_date = (!empty($val['end_date']) ? date('Y-m-d',$val['end_date']) : '');
				
				$dates = '
				<td>
					<div class="flex gap20">
						<div>
							<div><input type="date" name="'.$id.'_sale_start_date" value="'.$startdate.'" /></div>
						</div>
						<div>
							<div><input type="date" name="'.$id.'_sale_end_date" value="'.$end_date.'" /></div>
						</div>
					</div>
				</td>
				';
				
				// set a minimum value which will trigger a visual alert of low price
				$lowprice_class= $lowprice_msg='';
				if( $val['saleprice'] < 2 ) {
					$lowprice = ' low-price-alert';
					$lowprice_msg = '<small><i>this price may be too low</i></small>';
				}
			?>
			<tr class="<?php echo $lowprice_class; ?>">
				<td><?php echo $val['itemtitle']; ?></td>
				<td><?php echo $val['regprice']; ?></td>
				<td>
					<div class="flex-col">
					<span><input type="number" name="items[<?php echo $id; ?>]" min="0" step="0.01" value="<?php echo $val['saleprice']; ?>" /></span>
					<span class="discount-clear dashicons dashicons-no" title="clear value"></span>
					<?php echo $lowprice_msg; ?>
					</div>
				</td>
				<?php echo $dates; ?>
			</tr>
			
			<?php } ?>
			<?php echo $row; ?>
		</table>
		<?php echo $formclose; ?>
		
		<?php
	}
}
```
### Result
![price-set](https://github.com/WebsiteDons/WordPress/assets/42153624/830dac10-087b-4572-824d-6bdeb56d183a)


**Confirmation before saving**

This step has individual fields to set prices
![confirm-view](https://github.com/WebsiteDons/WordPress/assets/42153624/ac7eff2c-feec-4393-ac96-3852537bf83f)

**Display any current discount**
This is the home view if there are discounted items

![current-discount](https://github.com/WebsiteDons/WordPress/assets/42153624/bb08e507-58db-4c9f-bc46-9b9f77ceedf4)






