```php
add_action( 'woocommerce_before_single_product', function() {
  wc_enqueue_js('
  let char_limit = 400;
  var element_class = ".woocommerce-product-details__short-description";
  let ellipses = "... ";
  let full_content = $(element_class).html();

  if( full_content.length > char_limit ) {
    let short_content = content.substring(0, char_limit);
    let short_html = "";

    short_html += "<div class=\'truncated\'>";
    short_html += short_content + ellipses;
    short_html += "<span class=\'read-more\'>Read more</span>
    short_html += "</div>";

    short_html += "<div class=\'truncated\' style=\'display:none\'>";
    short_html += full_content;
    short_html += "<span class=\'read-less\'>Read less</span>";
    short_html += "</div>";
    // output
    $(element_class).html(short_html);
  }

  $(".read-more").click(function() {
    $(element_class+" .truncated").toggle();
  });
  $(".read-less").click(function() {
    $(element_class+" .truncated").toggle();
  });
  ');
}
```
