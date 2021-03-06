# Management via WP-CLI

WP-CLI is a powerful tool for managing sites and is a tool that we imagine many hosting partners will use. Below we will provide some information that we hope will make integration simpler for hosts that use WP-CLI.

- [Setting up WooCommerce and extensions after calling /register](#setting-up-woocommerce-and-extensions-after-calling-register)
- [Considerations for WP-ClI](#considerations-for-using-wp-cli)

## Setting up WooCommerce and extensions after calling /register

On a successful `/register` call, a hosting partner will end up with something like this:

```json
{
  "customer_id":1872797,
  "order_id":2666930,
  "access_token":"e95a8cc87027ec3c303e904d2ea69dd8e0ad0acf",
  "access_token_secret":"f71287fdce053102c71536b1648f27657e8be912",
  "site_id":42506
}
```

At this point, we have registered the site and made a purchase on WooCommerce.com, but we still need to save the values from the `/register` response as well as install/activate products on the site. Below is an example of how we might do that:

```shell
~ wp plugin install woocommerce --activate
Installing WooCommerce (3.5.6)
Downloading installation package from https://downloads.wordpress.org/plugin/woocommerce.3.5.6.zip...
Using cached file '/srv/users/user0c3505ec/.wp-cli/cache/plugin/woocommerce-3.5.6.zip'...
Unpacking the package...
Installing the plugin...
Plugin installed successfully.
Activating 'woocommerce'...
Plugin 'woocommerce' activated.
Success: Installed 1 of 1 plugins.
~ wp eval 'require_once WC()->plugin_path() . "/includes/admin/helper/class-wc-helper-options.php"; WC_Helper_Options::update( "auth", array( "access_token" => "e95a8cc87027ec3c303e904d2ea69dd8e0ad0acf", "access_token_secret" => "f71287fdce053102c71536b1648f27657e8be912", "site_id" => 42506, "user_id" => 1, "updated" => time(), ) );'
~ wp plugin install 'http://woothemes-products.s3.amazonaws.com/plugin-packages/woocommerce-shipping-usps/woocommerce-shipping-usps.zip?AWSAccessKeyId=AKIAJE6A7GBT4ZRLENMA&Expires=1544211649&Signature=lbxifNZRLGSresGFSE6wNoVyQ2w%3D' --activate
Downloading installation package from http://woothemes-products.s3.amazonaws.com/plugin-packages/woocommerce-shipping-usps/woocommerce-shipping-usps.zip?AWSAccessKeyId=AKIAJE6A7GBT4ZRLENMA&Expires=1544211649&Signature=lbxifNZRLGSresGFSE6wNoVyQ2w%3D...
Unpacking the package...
Installing the plugin...
Plugin installed successfully.
Activating 'woocommerce-shipping-usps'...
Plugin 'woocommerce-shipping-usps' activated.
Success: Installed 1 of 1 plugins.
```

After running the above commands, you should notice:

- The site has WooCommerce installed
- The USPS Shipping extension is installed and shows it's using a license in the WooCommerce.com subscriptions page

## Considerations for using WP-CLI

Due to some code not being loaded in the WP-CLI context, there are some issues that require special consideration to work around until fixed in core. We'll list these below along with suggestions for fixing.

### Setting authentication information

As part of the `/register` request, a hosting partner will get back information that allows the site to authenticate with WooCommerce.com. This information is expected to be set in the `woocommerce_helper_data` option, in an array, under the `auth` key. Core WooCommerce provides a helper to set this information, which simplifies setting the information to something like:

```php
<?php
WC_Helper_Options::update(
  'auth',
  array(
    'access_token'        => $response['access_token'],
    'access_token_secret' => $response['access_token_secret'],
    'site_id'             => $response['site_id'],
    'user_id'             => $user_id, // This is the ID of the user on the customer's site, which is usually 1.
    'updated'             => time(),
  )
);
```

When run in WP-CLI though, `WC_Helper_Options` is not available. A work-around for this is to require the file for the `WC_Helper_Options` class and then set the option. Something like this:

```
wp eval 'require_once WC()->plugin_path() . "/includes/admin/helper/class-wc-helper-options.php"; WC_Helper_Options::update( "auth", array( "access_token" => "ACCESS_TOKEN_HERE", "access_token_secret" => "ACCESS_TOKEN_SECRET_HERE", "site_id" => 123456789, "user_id" => 1, "updated" => time(), ) );'
```

### Automatically activating and deactivating subscriptions

There is a known, and [logged](https://github.com/woocommerce/woocommerce/issues/22762), issue where WooCommerce subscriptions for extensions are not automatically managed when the extension plugins are activated or deactivated. Until this is fixed in WooCommerce core, we suggest adding the following snippet of code that runs in an `mu-plugin`:

```php
<?php

class WooCommerce_CLI_Integration_Hosting_Partners {
  /**
   * The singleton instance if there is one.
   *
   * @var WooCommerce_CLI_Integration_Hosting_Partners
   **/
  private static $instance = null;

  /**
   * Enforces singleton pattern by returning a new instance or getting the existing instance.
   *
   * @return WooCommerce_CLI_Integration_Hosting_Partners
   */
  public static function init() {
    if ( is_null( self::$instance ) ) {
      self::$instance = new WooCommerce_CLI_Integration_Hosting_Partners();
    }

    return self::$instance;
  }

  /**
   * Constructor which adds hooks when the singleton is instantiated.
   */
  private function __construct() {
    add_action( 'activated_plugin', array( $this, 'activated_plugin' ) );
    add_action( 'deactivated_plugin', array( $this, 'deactivated_plugin' ) );
  }

  /**
   * Given a plugin filename, will attempt to activate a subscription on WooCommerce.com for the
   * plugin if it is a WooCommerce extension.
   *
   * @param string $filename The plugin filename.
   * @return void
   */
  public function activated_plugin( $filename ) {
    if ( ! $this->require_helper_files() ) {
      return;
    }
    WC_Helper::activated_plugin( $filename );
  }

  /**
   * Given a plugin filename, will attempt to deactivate a subscription on WooCommerce.com for the
   * plugin if it is a WooCommerce extension.
   *
   * @param string $filename The plugin filename.
   * @return void
   */
  public function deactivated_plugin( $filename ) {
    if ( ! $this->require_helper_files() ) {
      return;
    }
    WC_Helper::deactivated_plugin( $filename );
  }

  /**
   * Will return true when WooCommerce is installed and active, or false otherwise.
   *
   * @return boolean
   */
  private function is_woo_active() {
    return class_exists( 'WooCommerce' ) && function_exists( 'WC' );
  }

  /**
   * Will require all helper files if WooCommerce is active.
   *
   * @return boolean
   */
  private function require_helper_files() {
    if ( ! $this->is_woo_active() ) {
      return false;
    }

    try {
      foreach ( glob( WC()->plugin_path() . '/includes/admin/helper/*.php' ) as $helper_file ) {
        require_once( $helper_file );
      }
    } catch ( Exception $e ) {
      return false;
    }

    return true;
  }
}

WooCommerce_CLI_Integration_Hosting_Partners::init();

```

Once this code is added in an `mu-plugin`, then subscriptions should be properly activated/deactivated whenever the plugin file for the extension is activated/deactivated.
