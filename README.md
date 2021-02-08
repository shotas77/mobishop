*/

class WC_Gateway_LibertyBank_Initer {
	const PERMALINK = 'prelibrebankinstallment';

	public function __construct() {
		add_action( 'plugins_loaded', 'init_libertybank_gateway_class' );
		add_action( 'init', array( $this, 'add_page') );
		add_filter( 'woocommerce_payment_gateways', array( $this, 'add_libertybank_gateway_class') );
		add_shortcode( 'libertybank_precheckout', array($this, 'showForm') );
	}

	public function add_page() {
		if (!get_page_by_path(self::PERMALINK)) {
				wp_insert_post(array(
					'post_type' => 'page',
					'post_title' => 'Libertybank preredirect page',
					'post_content' => '[libertybank_precheckout]',
					'post_name' => self::PERMALINK,
					'post_status' => 'publish'
			));
		}
	}

	public function add_libertybank_gateway_class( $methods ) {
		$methods[] = 'WC_Gateway_LibertyBank_Installments'; 
		return $methods;
	}

	public function showForm() {
		global $wp;
		$order_id = $_GET['order_id'];
		if (!$order_id) {
			$order_id = $wp->query_vars['page'];
		}
		if (!$order_id) {
			return '';
		}
		$order = wc_get_order($order_id);
		if (!$order) {
			return '';
		}
		$callid = urldecode($order->order_key);

		$settings_key = 'woocommerce_libertybankinstallments_settings'; // TODO: remove hardcode
		$settings = get_option($settings_key);

		$secretkey = $settings['secretkey'];
		$merchant  = $settings['merchant'];
		$testmode  = $settings['testmode'] === 'yes' ? 1 : 0;
		$ordercode = $order->id;
		$callid = $order->order_key;
		$shipping_address = $order->get_formatted_shipping_address();
		$installmenttype = 0;
		$products = $order->get_items();

		$str = $secretkey
			. $merchant
			. $ordercode
			. $callid
			. $shipping_address
			. $testmode;

		foreach ($products as $key => $product) {
			$str .= $key.$product['name'].$product['qty'].$product['line_total'].$product['type'].$installmenttype; 
		}         
		$check = strtoupper(hash('sha256', $str));
		return $this->getHTMLTemplate($products, $merchant, $testmode, $ordercode, $callid, $shipping_address, $installmenttype, $check);
	}
