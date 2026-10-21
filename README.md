# Beltoft Cache Support

WordPress + WooCommerce companion to an **nginx FastCGI full-page cache**. With it, every anonymous page can
be served from one shared cached copy, including to shoppers with something in their cart, without leaking one
visitor's cart to the next. It also purges and re-warms the cache when content changes.

The plugin is one file, `beltoft-cache-support.php`. It has no settings screen: the setup is nginx, a cron line
and, if your layout differs, a few constants. Its doc blocks explain why it works the way it does. This README
covers how to set it up.

## What it does

- **Cart header under a shared cache.**
  - nginx never *stores* a response rendered for someone holding a cart, so the cached copy is always the
    cart-less one.
  - nginx still *serves* that copy to cart holders. Their header count and subtotal are painted before first
    paint from a readable cookie, `beltoft_cart_summary`, which the plugin keeps in step with the cart.
  - WooCommerce's cart fragments still load afterwards and correct anything stale.
- **Login from a cached page.** A cached WooCommerce login form carries a nonce made for "no session", which
  WooCommerce silently rejects once the visitor has a cart. The plugin provides
  `GET /wp-json/beltoft/v1/login-nonce`, which returns a fresh `woocommerce-login` nonce for the caller's own
  session. The plugin does **not** change any form: a login form shown on cached pages needs a small script that
  fetches this and replaces the form's `woocommerce-login-nonce` value just before submitting. See "For other
  code".
- **Purging.** nginx here has no purge module, so a purge deletes the cache file. nginx names each file after
  the md5 of its cache key. The cache is purged when:
  - published content changes: save, trash, untrash, unpublish or delete of a `product`, `product_variation`,
    `page`, `post` or `bricks_template` (filter `beltoft_cache_purge_post_types`). Bricks builder saves count;
  - menus, the theme, or Bricks global classes, theme styles, variables, palette or settings change;
  - a comment becomes visible: approved on posting, or moved into or out of approved. Merely posting one doesn't
    purge, so nobody can empty the cache by submitting reviews;
  - anything fires `do_action( 'beltoft_cache_flush' )`;
  - someone uses the admin bar ("Limpar cache de páginas" for everything, or the current page only).
- **Warming.**
  - After a purge, the changed page, the front page and the shop are re-fetched right after the response.
  - A minute later a **sweep** re-fetches every other preloadable URL, one at a time, so it never holds more than
    one PHP worker. A newer purge stops the running sweep and queues its own.
  - Warm-ups send no cookies, so they always render the anonymous page.
  - Cart, checkout, account and similar paths are never preloaded (`BELTOFT_CACHE_PRELOAD_SKIP`).

## Requirements

- nginx with `fastcgi_cache`, serving WordPress through PHP-FPM. **No** `levels=` on the cache path: the
  plugin expects the cache files directly in the directory.
- WordPress, WooCommerce, PHP 8.0+, WP-CLI.
- The PHP-FPM user (normally `www-data`) must be able to **delete** files in the cache directory.
- The server must be able to fetch its own public URLs over HTTPS, for warming. Warm-ups use a browser
  User-Agent, because bot filters that block library agents would otherwise make every warm-up fail.

## Setup

### 1. nginx

Cache zone and skip maps, at `http` level (for example at the top of the site's vhost file):

```nginx
# The skip maps push the variable count past the default hash size.
variables_hash_max_size 2048;

# One directory per site, NO levels= (the plugin deletes entries by their md5 name).
fastcgi_cache_path /var/www/cache/example.com
	keys_zone=SITE_PAGE:50m inactive=24h max_size=1024m use_temp_path=off;

# Only the canonical hostname is cacheable.
map $host $page_cache_host_skip {
	default     1;
	example.com 0;
}

map $request_method $page_cache_method_skip {
	default 1;
	GET     0;
}

# Query strings are dynamic and never cached.
map $args $page_cache_query_skip {
	default 1;
	""      0;
}

# Admin, API, transactional and personal pages never use the cache. Use your own slugs.
map $request_uri $page_cache_uri_skip {
	default 0;
	"~*^/(?:[a-z][a-z]/)?(?:wp-admin(?:/|$)|wp-login\.php$|wp-json(?:/|$)|wp-cron\.php$|xmlrpc\.php$|wc-api(?:/|$)|wc-auth(?:/|$))" 1;
	"~*^/(?:[a-z][a-z]/)?(?:cart|checkout|my-account|wishlist)(?:/|$)" 1;
}

# Logged-in, comment-author and password-post visitors neither read nor fill the cache.
map $http_cookie $page_cache_auth_skip {
	default 0;
	~*wordpress_(?:logged_in|sec)_[^=]*= 1;
	~*comment_author_[^=]*=               1;
	~*wp-postpass_[^=]*=                  1;
}

# Cart holders may READ the cached copy, but a response rendered for them is never STORED.
map $http_cookie $page_cache_store_skip {
	default 0;
	~*(?:woocommerce_items_in_cart|woocommerce_cart_hash|wp_woocommerce_session_[^=]*)= 1;
}
```

In the `server` block, a header that shows the cache result (HIT / MISS / BYPASS). If you add any `add_header` at
server level, repeat your security headers there too: nginx stops inheriting `http`-level `add_header`s as soon
as a level defines its own.

```nginx
add_header X-Cache $upstream_cache_status always;
```

In the PHP `location`:

```nginx
fastcgi_cache SITE_PAGE;
fastcgi_cache_key "$scheme$request_method$host$request_uri";
fastcgi_cache_methods GET;
fastcgi_cache_bypass
	$page_cache_host_skip $page_cache_method_skip $page_cache_query_skip
	$page_cache_uri_skip $page_cache_auth_skip;
fastcgi_no_cache
	$page_cache_host_skip $page_cache_method_skip $page_cache_query_skip
	$page_cache_uri_skip $page_cache_auth_skip
	$page_cache_store_skip
	$upstream_http_set_cookie;
fastcgi_cache_valid 200 12h;
fastcgi_cache_lock on;
fastcgi_cache_lock_timeout 15s;
fastcgi_cache_lock_age 15s;
fastcgi_cache_revalidate on;
fastcgi_cache_use_stale error timeout invalid_header updating http_500 http_503;
```

What the rules guarantee:
- `$page_cache_store_skip` and `$upstream_http_set_cookie` in `fastcgi_no_cache` are the safety guarantee: nothing
  rendered for a session, and nothing that sets a cookie, is ever stored.
- `$page_cache_store_skip` is deliberately **not** in `fastcgi_cache_bypass`, so cart holders still get hits.
- `fastcgi_cache_valid 200 12h` should match `BELTOFT_CACHE_TTL`.

Then `nginx -t && systemctl reload nginx`.

### 2. Create the cache directory

```bash
mkdir -p /var/www/cache/example.com
chown www-data:www-data /var/www/cache/example.com
```

### 3. Match the plugin to nginx

The plugin works out the cache file of a URL by itself. It only finds the right file if three values match
nginx exactly:

| Constant | Default | Must equal |
|---|---|---|
| `BELTOFT_CACHE_DIR` | `/var/www/cache/<name of the WordPress folder>` | the `fastcgi_cache_path` directory |
| `BELTOFT_CACHE_KEY_SCHEME` | `http` | `$scheme` as **this** nginx sees it |
| `BELTOFT_CACHE_KEY_METHOD` | `GET` | `$request_method` of cached requests |
| `BELTOFT_CACHE_TTL` | 12 hours | `fastcgi_cache_valid 200 …` |

The scheme is `http` because TLS ends at a proxy in front of nginx, and nginx listens on port 80. If your nginx
terminates TLS itself, define `BELTOFT_CACHE_KEY_SCHEME` as `https`. Override any of these in `wp-config.php`,
above the "stop editing" line:

```php
define( 'BELTOFT_CACHE_DIR', '/var/www/cache/example.com' );
define( 'BELTOFT_CACHE_KEY_SCHEME', 'https' );
```

**A mismatch fails silently.** Purges miss their files and pages go stale until they expire. Step 6 is how you
catch it.

### 4. Install and activate

Copy the plugin folder to `wp-content/plugins/beltoft-cache-support/`, then
`wp plugin activate beltoft-cache-support`.

### 5. Cron

WP-Cron should be run by the system (`define( 'DISABLE_WP_CRON', true );` plus a crontab line running
`wp cron event run --due-now` every minute). Add a **dedicated sweep runner** in the web user's crontab
(`crontab -u www-data -e`):

```cron
*/1 * * * * /usr/local/bin/wp beltoft-cache sweep --path=/var/www/example.com --quiet 2>/dev/null
```

It's optional but recommended. Without it, sweeps run inside the shared WP-Cron process, whose ~25 s batches
delay other scheduled work there, such as Action Scheduler's queued emails. When the dedicated runner hasn't been
seen for 5 minutes, sweeps fall back to WP-Cron by themselves.

### 6. Check

```bash
sudo -u www-data wp beltoft-cache status --path=/var/www/example.com
```

Open the front page once in a private window first, so something is cached, then check the output:
- `Key check: OK - key matches a cached entry`. Anything else means step 3 doesn't match nginx.
- `Warm: N of M preloadable URLs`, which grows after a purge as the sweep runs.

Then from the shell, with a **GET**. A `HEAD` request (`curl -I`) always shows BYPASS, because only GET is
cached:

```bash
curl -s -o /dev/null -D - https://example.com/ | grep -i x-cache    # MISS, then HIT on the second request
```

And in a browser: add a product to the cart and browse. Pages should stay HIT (check `X-Cache` in devtools)
while the header shows the right count and subtotal. If a WooCommerce login form appears on cached pages, log in from it with the cart still
full: it must work (see the login nonce note under "For other code").

## Commands

```bash
wp beltoft-cache status           # cache dir, key basis, entries, key check, warm coverage, generation, last warm
wp beltoft-cache purge            # delete every entry, then warm (in the shell, so you see it)
wp beltoft-cache purge-url <url>  # one URL; must be on the canonical host and currently cached
wp beltoft-cache preload          # warm now
wp beltoft-cache sweep            # run the queued sweep (what the cron line calls)
```

## For other code

- `beltoft_cache_purge_url( $url )`: purge one URL, for example a page that shows another product's stock.
- `beltoft_cache_purge_all()`: purge everything, and schedule the sweep.
- `do_action( 'beltoft_cache_flush' )`: the same, without depending on the function existing.
- `beltoft_cache_warm_after_response( $post_id )`: re-fetch a post's page after the current response.

Guard calls with `function_exists()`, so your code still works when this plugin is inactive.

Filters:
- `beltoft_cache_purge_post_types`: the post types whose changes purge.
- `beltoft_cache_count_badges`: header count badges that exist only when the cart has items and must be created
  client-side. Each one is `array( 'wrap' => '<css selector>', 'id' => '<element id>', 'class' => '<classes>' )`.
  The default is the Bricks mini-cart badge `#brxe-mccntf` > `#brxe-mccnt1`.

Login forms on cached pages: before submit, fetch `/wp-json/beltoft/v1/login-nonce` (same origin, credentials
included) and write the returned `nonce` into the form's `woocommerce-login-nonce` field. Without this, a shopper
with a cart can't log in from a cached page: the page just reloads, with no error.

Header markup:
- Counts in `span.cart-count` and subtotals in `span.cart-subtotal` (WooCommerce's fragment selectors) are
  painted from the cookie.
- A Bricks header needs its badge registered through `beltoft_cache_count_badges` if its IDs differ from the
  default.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Pages stay stale after saving | `status` key check fails. The dir, scheme or method doesn't match nginx (step 3), or the cache path has `levels=`. |
| `X-Cache` is always BYPASS | You're testing with HEAD, or carry a logged-in, comment or `wp-postpass` cookie, or the URL has a query string or matches the URI skip map. |
| `X-Cache` is always MISS | The response sets a cookie (`$upstream_http_set_cookie`). Find the plugin or theme that sets one on every page. |
| Header shows 0 items on a cached page with a full cart | The badge isn't in a painted target: register it with `beltoft_cache_count_badges`, or use `span.cart-count`. |
| Login on a cached page just reloads for shoppers | The form posts the cached nonce. Add the nonce fetch (see "For other code"), and make sure `/wp-json/beltoft/v1/login-nonce` reaches PHP uncached. |
| Warm-ups fail, `Warm` never grows | The server can't fetch its own HTTPS URLs, or a firewall or bot filter blocks it. |
| Scheduled emails go out slowly after purges | The dedicated sweep runner (step 5) is missing. |

## Authorship

Written by beltoft.net.
