# GitHub Theme Updater — PHP Module

A drop-in WordPress module that checks for theme updates from a GitHub repository and integrates with the WordPress core update system. Users see a dashboard widget with version info and can update with one click.

## Placeholder reference

| Placeholder | Example | Description |
|---|---|---|
| `{{THEME_SLUG}}` | `my-theme` | Theme directory name |
| `{{THEME_NAME}}` | `My Theme` | Human-readable theme name |
| `{{REPO_OWNER}}` | `johndoe` | GitHub username or org |
| `{{REPO_NAME}}` | `my-theme` | GitHub repository name |
| `{{TEXT_DOMAIN}}` | `my-theme` | WordPress text domain |
| `{{PREFIX}}` | `my_theme` | Snake_case prefix for functions/options |
| `{{PREFIX_UPPER}}` | `MY_THEME` | Uppercase prefix for constants |

## File structure

```
includes/github-updater/
├── loader.php            # Always
├── client.php            # Always
├── updater.php           # Always
├── dashboard-widget.php  # Always
├── settings.php          # Private repos only
└── admin-notice.php      # Private repos only
```

## Public vs. private repos

- **Public repos**: The GitHub API `/releases/latest` endpoint works without authentication (60 requests/hour rate limit per IP). With 6-hour caching, this is more than sufficient. No token, no settings page, no admin notice needed.
- **Private repos**: A GitHub Personal Access Token is required. The module provides a settings page (Appearance → Theme Updates) and an admin notice guiding the admin to configure it. Token can be defined as a PHP constant in `wp-config.php` (recommended) or saved in the database via the settings page.

The client code handles both cases: if a token is available, it's used; if not (public repo), requests are unauthenticated.

---

## File 1: loader.php

```php
<?php
/**
 * GitHub Theme Updater Loader
 *
 * Loads all GitHub updater components in the correct order.
 */

if (!defined('ABSPATH')) {
    exit;
}

class {{PREFIX}}_GitHub_Updater_Loader {
    public function __construct() {
        $this->load_components();
    }

    private function load_components() {
        $base_path = get_stylesheet_directory() . '/includes/github-updater/';

        $files = array(
            'client.php',
            'updater.php',
            'dashboard-widget.php',
        );

        // PRIVATE_REPO_ONLY: Add these files for private repositories
        $files[] = 'settings.php';
        $files[] = 'admin-notice.php';
        // END PRIVATE_REPO_ONLY

        foreach ($files as $file) {
            $file_path = $base_path . $file;
            if (file_exists($file_path)) {
                require_once $file_path;
            }
        }
    }
}

function {{PREFIX}}_init_github_updater() {
    new {{PREFIX}}_GitHub_Updater_Loader();
}
add_action('plugins_loaded', '{{PREFIX}}_init_github_updater', 5);
```

For **public repos**, remove the lines between `PRIVATE_REPO_ONLY` comments.

---

## File 2: client.php

```php
<?php
/**
 * GitHub Theme Update Client
 *
 * Handles communication with the GitHub API for theme updates.
 */

if (!defined('ABSPATH')) {
    exit;
}

class {{PREFIX}}_GitHub_Client {

    const REPO_OWNER = '{{REPO_OWNER}}';
    const REPO_NAME  = '{{REPO_NAME}}';
    const CACHE_KEY  = '{{PREFIX}}_github_latest_release';
    const CACHE_TTL  = 6 * HOUR_IN_SECONDS;

    /**
     * Get the GitHub API token.
     *
     * Priority: wp-config.php constant > database option.
     * Returns false if no token is configured (acceptable for public repos).
     */
    public function get_token() {
        if (defined('{{PREFIX_UPPER}}_GITHUB_TOKEN') && {{PREFIX_UPPER}}_GITHUB_TOKEN) {
            return {{PREFIX_UPPER}}_GITHUB_TOKEN;
        }

        $token = get_option('{{PREFIX}}_github_token', '');
        return !empty($token) ? $token : false;
    }

    /**
     * Check if the token was defined via wp-config.php constant.
     */
    public function is_token_from_constant() {
        return defined('{{PREFIX_UPPER}}_GITHUB_TOKEN') && {{PREFIX_UPPER}}_GITHUB_TOKEN;
    }

    /**
     * Check if a token is configured (relevant for private repos).
     * For public repos, this may return false and that's OK.
     */
    public function is_configured() {
        return $this->get_token() !== false;
    }

    /**
     * Get the latest release from GitHub.
     *
     * @param bool $force_refresh Skip cache and fetch from API.
     * @return array|WP_Error Release data or error.
     */
    public function get_latest_release($force_refresh = false) {
        if (!$force_refresh) {
            $cached = get_transient(self::CACHE_KEY);
            if ($cached !== false) {
                return $cached;
            }
        }

        $url = sprintf(
            'https://api.github.com/repos/%s/%s/releases/latest',
            self::REPO_OWNER,
            self::REPO_NAME
        );

        $headers = array(
            'Accept'     => 'application/vnd.github.v3+json',
            'User-Agent' => 'WordPress/' . get_bloginfo('version'),
        );

        // Add auth header if token is available
        $token = $this->get_token();
        if ($token) {
            $headers['Authorization'] = 'Bearer ' . $token;
        }

        $response = wp_remote_get($url, array(
            'timeout' => 30,
            'headers' => $headers,
        ));

        if (is_wp_error($response)) {
            $this->log_error('Request failed: ' . $response->get_error_message());
            return $response;
        }

        $code = wp_remote_retrieve_response_code($response);
        if ($code !== 200) {
            $this->log_error("API error: HTTP $code");
            return new WP_Error(
                'api_error',
                sprintf(__('GitHub API error: %s', '{{TEXT_DOMAIN}}'), $code)
            );
        }

        $data = json_decode(wp_remote_retrieve_body($response), true);

        if (json_last_error() !== JSON_ERROR_NONE) {
            return new WP_Error('json_error', __('Error parsing GitHub response.', '{{TEXT_DOMAIN}}'));
        }

        $release = array(
            'tag_name'     => isset($data['tag_name']) ? $data['tag_name'] : '',
            'name'         => isset($data['name']) ? $data['name'] : '',
            'body'         => isset($data['body']) ? $data['body'] : '',
            'published_at' => isset($data['published_at']) ? $data['published_at'] : '',
            'html_url'     => isset($data['html_url']) ? $data['html_url'] : '',
            'zipball_url'  => isset($data['zipball_url']) ? $data['zipball_url'] : '',
        );

        set_transient(self::CACHE_KEY, $release, self::CACHE_TTL);

        return $release;
    }

    /**
     * Get download URL for a specific tag or the latest release.
     */
    public function get_download_url($tag = '') {
        if (empty($tag)) {
            $release = $this->get_latest_release();
            if (is_wp_error($release)) {
                return '';
            }
            return $release['zipball_url'];
        }

        return sprintf(
            'https://api.github.com/repos/%s/%s/zipball/%s',
            self::REPO_OWNER,
            self::REPO_NAME,
            $tag
        );
    }

    public function clear_cache() {
        delete_transient(self::CACHE_KEY);
    }

    /**
     * Get human-readable last check time.
     */
    public function get_last_check_time() {
        $timeout = get_option('_transient_timeout_' . self::CACHE_KEY);
        if (!$timeout) {
            return false;
        }
        $set_time = $timeout - self::CACHE_TTL;
        return date_i18n(get_option('date_format') . ' ' . get_option('time_format'), $set_time);
    }

    private function log_error($message) {
        if (defined('WP_DEBUG') && WP_DEBUG) {
            $token = $this->get_token();
            if ($token) {
                $message = str_replace($token, 'REDACTED', $message);
            }
            error_log('[{{THEME_NAME}} Updater] Error: ' . $message);
        }
    }
}

/**
 * Singleton accessor for the GitHub client.
 */
function {{PREFIX}}_github_client() {
    static $client = null;
    if ($client === null) {
        $client = new {{PREFIX}}_GitHub_Client();
    }
    return $client;
}
```

---

## File 3: updater.php

```php
<?php
/**
 * GitHub Theme Updater
 *
 * Integrates with the WordPress core update system to show theme updates
 * from the GitHub repository on the Dashboard and Appearance → Themes.
 */

if (!defined('ABSPATH')) {
    exit;
}

class {{PREFIX}}_GitHub_Updater {

    private $theme_slug = '{{THEME_SLUG}}';

    public function __construct() {
        add_filter('pre_set_site_transient_update_themes', array($this, 'check_for_update'));
        add_filter('upgrader_pre_download', array($this, 'pre_download'), 10, 3);
        add_filter('upgrader_source_selection', array($this, 'fix_source_dir'), 10, 4);
    }

    /**
     * Inject update data into the WordPress update transient.
     */
    public function check_for_update($transient) {
        if (empty($transient->checked)) {
            return $transient;
        }

        $client = {{PREFIX}}_github_client();
        $release = $client->get_latest_release();

        if (is_wp_error($release)) {
            return $transient;
        }

        $current_version = wp_get_theme($this->theme_slug)->get('Version');
        $remote_version  = ltrim($release['tag_name'], 'v');

        if (version_compare($remote_version, $current_version, '>')) {
            $transient->response[$this->theme_slug] = array(
                'theme'       => $this->theme_slug,
                'new_version' => $remote_version,
                'url'         => $release['html_url'],
                'package'     => $release['zipball_url'],
            );
        } else {
            $transient->no_update[$this->theme_slug] = array(
                'theme'       => $this->theme_slug,
                'new_version' => $current_version,
                'url'         => '',
                'package'     => '',
            );
        }

        return $transient;
    }

    /**
     * Handle authenticated downloads from private repositories.
     *
     * For public repos, this method returns early and WordPress handles
     * the download normally. For private repos, it adds the auth header.
     */
    public function pre_download($reply, $package, $upgrader) {
        if (strpos($package, 'api.github.com') === false) {
            return $reply;
        }
        if (strpos($package, {{PREFIX}}_GitHub_Client::REPO_NAME) === false) {
            return $reply;
        }

        $client = {{PREFIX}}_github_client();
        $token  = $client->get_token();

        // Public repo — let WordPress handle the download
        if (!$token) {
            return $reply;
        }

        // Private repo — download with authentication
        $response = wp_remote_get($package, array(
            'timeout'  => 300,
            'stream'   => true,
            'filename' => $this->get_temp_file(),
            'headers'  => array(
                'Authorization' => 'Bearer ' . $token,
                'Accept'        => 'application/vnd.github.v3+json',
                'User-Agent'    => 'WordPress/' . get_bloginfo('version'),
            ),
        ));

        if (is_wp_error($response)) {
            return $response;
        }

        if (wp_remote_retrieve_response_code($response) !== 200) {
            return new WP_Error(
                'download_error',
                __('Download failed.', '{{TEXT_DOMAIN}}')
            );
        }

        return $this->get_temp_file();
    }

    /**
     * Fix directory name after ZIP extraction.
     *
     * GitHub zipballs extract to "owner-repo-hash/" but WordPress
     * expects the theme directory name.
     */
    public function fix_source_dir($source, $remote_source, $upgrader, $hook_extra) {
        if (!isset($hook_extra['theme']) || $hook_extra['theme'] !== $this->theme_slug) {
            return $source;
        }

        global $wp_filesystem;

        $source_base = basename($source);
        if (strpos($source_base, {{PREFIX}}_GitHub_Client::REPO_OWNER) === 0) {
            $corrected = trailingslashit(dirname($source)) . $this->theme_slug . '/';
            if ($wp_filesystem->move($source, $corrected, true)) {
                return $corrected;
            }
            return new WP_Error('rename_failed', __('Could not rename theme directory.', '{{TEXT_DOMAIN}}'));
        }

        return $source;
    }

    private function get_temp_file() {
        return trailingslashit(wp_upload_dir()['basedir']) . '{{PREFIX}}-github-update.zip';
    }
}

new {{PREFIX}}_GitHub_Updater();
```

---

## File 4: dashboard-widget.php

```php
<?php
/**
 * GitHub Theme Updater Dashboard Widget
 *
 * Shows current vs. latest version and update controls on the WordPress dashboard.
 */

if (!defined('ABSPATH')) {
    exit;
}

function {{PREFIX}}_register_dashboard_widget() {
    if (!current_user_can('update_themes')) {
        return;
    }

    wp_add_dashboard_widget(
        '{{PREFIX}}_github_updater_widget',
        __('{{THEME_NAME}} Updates', '{{TEXT_DOMAIN}}'),
        '{{PREFIX}}_dashboard_widget_render'
    );
}
add_action('wp_dashboard_setup', '{{PREFIX}}_register_dashboard_widget');

function {{PREFIX}}_dashboard_widget_render() {
    $client          = {{PREFIX}}_github_client();
    $theme           = wp_get_theme('{{THEME_SLUG}}');
    $current_version = $theme->get('Version');

    // PRIVATE_REPO_ONLY_START
    if (!$client->is_configured()) {
        ?>
        <p class="description">
            <?php _e('GitHub token is not configured. Automatic updates are disabled.', '{{TEXT_DOMAIN}}'); ?>
        </p>
        <p>
            <a href="<?php echo esc_url(admin_url('themes.php?page={{PREFIX}}-github-updater')); ?>" class="button">
                <?php _e('Configure', '{{TEXT_DOMAIN}}'); ?>
            </a>
        </p>
        <?php
        return;
    }
    // PRIVATE_REPO_ONLY_END

    $release          = $client->get_latest_release();
    $has_error        = is_wp_error($release);
    $latest_version   = $has_error ? null : ltrim($release['tag_name'], 'v');
    $update_available = $latest_version && version_compare($latest_version, $current_version, '>');
    $last_check       = $client->get_last_check_time();
    ?>

    <div id="{{PREFIX}}-updater-widget-content">
        <table class="widefat" style="border: none; box-shadow: none;">
            <tbody>
                <tr>
                    <td><strong><?php _e('Current version:', '{{TEXT_DOMAIN}}'); ?></strong></td>
                    <td><code><?php echo esc_html($current_version); ?></code></td>
                </tr>
                <tr>
                    <td><strong><?php _e('Latest version:', '{{TEXT_DOMAIN}}'); ?></strong></td>
                    <td>
                        <?php if ($has_error): ?>
                            <span style="color: #b32d2e;">
                                <?php _e('Error fetching release info', '{{TEXT_DOMAIN}}'); ?>
                            </span>
                        <?php elseif ($update_available): ?>
                            <code style="background: #d63638; color: white; padding: 2px 6px;"><?php echo esc_html($latest_version); ?></code>
                            <span class="dashicons dashicons-warning" style="color: #d63638;"></span>
                        <?php else: ?>
                            <code style="background: #00a32a; color: white; padding: 2px 6px;"><?php echo esc_html($latest_version); ?></code>
                            <span class="dashicons dashicons-yes-alt" style="color: #00a32a;"></span>
                        <?php endif; ?>
                    </td>
                </tr>
                <?php if ($last_check): ?>
                <tr>
                    <td><strong><?php _e('Last checked:', '{{TEXT_DOMAIN}}'); ?></strong></td>
                    <td><?php echo esc_html($last_check); ?></td>
                </tr>
                <?php endif; ?>
            </tbody>
        </table>

        <p style="margin-top: 12px;">
            <?php if ($update_available): ?>
                <a href="<?php echo esc_url(admin_url('update-core.php')); ?>" class="button button-primary">
                    <span class="dashicons dashicons-update" style="margin-top: 4px;"></span>
                    <?php _e('Update now', '{{TEXT_DOMAIN}}'); ?>
                </a>
            <?php endif; ?>

            <button type="button" class="button" id="{{PREFIX}}-updater-check-btn">
                <span class="dashicons dashicons-search" style="margin-top: 4px;"></span>
                <?php _e('Check for updates', '{{TEXT_DOMAIN}}'); ?>
            </button>

            <?php if (!$has_error && !empty($release['html_url'])): ?>
                <a href="<?php echo esc_url($release['html_url']); ?>" target="_blank" class="button"
                   title="<?php esc_attr_e('View on GitHub', '{{TEXT_DOMAIN}}'); ?>">
                    <span class="dashicons dashicons-external" style="margin-top: 4px;"></span>
                </a>
            <?php endif; ?>
        </p>

        <div id="{{PREFIX}}-updater-spinner" style="display: none; margin-top: 10px;">
            <span class="spinner is-active" style="float: none; margin: 0;"></span>
            <?php _e('Checking...', '{{TEXT_DOMAIN}}'); ?>
        </div>

        <div id="{{PREFIX}}-updater-message" style="margin-top: 10px;"></div>
    </div>

    <script>
    jQuery(document).ready(function($) {
        $('#{{PREFIX}}-updater-check-btn').on('click', function() {
            var $btn = $(this);
            var $spinner = $('#{{PREFIX}}-updater-spinner');
            var $message = $('#{{PREFIX}}-updater-message');

            $btn.prop('disabled', true);
            $spinner.show();
            $message.html('');

            $.post(ajaxurl, {
                action: '{{PREFIX}}_check_update',
                nonce: '<?php echo wp_create_nonce('{{PREFIX}}_check_update'); ?>'
            }, function(response) {
                $spinner.hide();
                $btn.prop('disabled', false);
                if (response.success) {
                    location.reload();
                } else {
                    $message.html('<div class="notice notice-error inline"><p>' + response.data + '</p></div>');
                }
            }).fail(function() {
                $spinner.hide();
                $btn.prop('disabled', false);
                $message.html('<div class="notice notice-error inline"><p><?php _e('Connection error', '{{TEXT_DOMAIN}}'); ?></p></div>');
            });
        });
    });
    </script>
    <?php
}

function {{PREFIX}}_ajax_check_update() {
    check_ajax_referer('{{PREFIX}}_check_update', 'nonce');

    if (!current_user_can('update_themes')) {
        wp_send_json_error(__('Insufficient permissions.', '{{TEXT_DOMAIN}}'));
    }

    $client = {{PREFIX}}_github_client();
    $client->clear_cache();
    $release = $client->get_latest_release(true);

    if (is_wp_error($release)) {
        wp_send_json_error($release->get_error_message());
    }

    wp_send_json_success();
}
add_action('wp_ajax_{{PREFIX}}_check_update', '{{PREFIX}}_ajax_check_update');
```

For **public repos**, remove the block between `PRIVATE_REPO_ONLY_START` and `PRIVATE_REPO_ONLY_END` comments. The widget will always show version info without requiring token configuration.

---

## File 5: settings.php (PRIVATE REPOS ONLY)

Skip this file entirely for public repositories.

```php
<?php
/**
 * GitHub Theme Updater Settings
 *
 * Adds a settings page under Appearance for configuring the GitHub token.
 */

if (!defined('ABSPATH')) {
    exit;
}

function {{PREFIX}}_updater_add_settings_page() {
    add_theme_page(
        __('Theme Updates', '{{TEXT_DOMAIN}}'),
        __('Theme Updates', '{{TEXT_DOMAIN}}'),
        'manage_options',
        '{{PREFIX}}-github-updater',
        '{{PREFIX}}_updater_render_settings_page'
    );
}
add_action('admin_menu', '{{PREFIX}}_updater_add_settings_page');

function {{PREFIX}}_updater_register_settings() {
    register_setting('{{PREFIX}}_updater_settings', '{{PREFIX}}_github_token', array(
        'type'              => 'string',
        'sanitize_callback' => 'sanitize_text_field',
        'default'           => '',
    ));
}
add_action('admin_init', '{{PREFIX}}_updater_register_settings');

function {{PREFIX}}_updater_render_settings_page() {
    $client            = {{PREFIX}}_github_client();
    $token_from_const  = $client->is_token_from_constant();
    $is_configured     = $client->is_configured();
    $current_version   = wp_get_theme('{{THEME_SLUG}}')->get('Version');
    $release           = $is_configured ? $client->get_latest_release() : null;
    $latest_version    = (!is_wp_error($release) && !empty($release['tag_name']))
        ? ltrim($release['tag_name'], 'v')
        : __('Unknown', '{{TEXT_DOMAIN}}');
    $last_check        = $client->get_last_check_time();

    // Handle manual check
    if (isset($_POST['{{PREFIX}}_check_now']) && check_admin_referer('{{PREFIX}}_updater_settings-options')) {
        $client->clear_cache();
        delete_site_transient('update_themes');
        $release = $client->get_latest_release(true);
        $latest_version = (!is_wp_error($release) && !empty($release['tag_name']))
            ? ltrim($release['tag_name'], 'v')
            : __('Unknown', '{{TEXT_DOMAIN}}');
        $last_check = $client->get_last_check_time();
        echo '<div class="notice notice-success is-dismissible"><p>' .
             __('Update check completed.', '{{TEXT_DOMAIN}}') . '</p></div>';
    }
    ?>
    <div class="wrap">
        <h1><?php _e('Theme Updates', '{{TEXT_DOMAIN}}'); ?></h1>

        <form method="post" action="options.php">
            <?php settings_fields('{{PREFIX}}_updater_settings'); ?>

            <h2><?php _e('GitHub Configuration', '{{TEXT_DOMAIN}}'); ?></h2>

            <table class="form-table">
                <tbody>
                    <?php if ($token_from_const): ?>
                    <tr>
                        <th scope="row"><?php _e('GitHub Token', '{{TEXT_DOMAIN}}'); ?></th>
                        <td>
                            <span class="dashicons dashicons-yes-alt" style="color: green;"></span>
                            <?php printf(
                                __('Configured via %s constant in wp-config.php', '{{TEXT_DOMAIN}}'),
                                '<code>{{PREFIX_UPPER}}_GITHUB_TOKEN</code>'
                            ); ?>
                        </td>
                    </tr>
                    <?php else: ?>
                    <tr>
                        <th scope="row">
                            <label for="{{PREFIX}}_github_token"><?php _e('GitHub Token', '{{TEXT_DOMAIN}}'); ?></label>
                        </th>
                        <td>
                            <input type="password" name="{{PREFIX}}_github_token" id="{{PREFIX}}_github_token"
                                   value="<?php echo esc_attr(get_option('{{PREFIX}}_github_token', '')); ?>"
                                   class="regular-text" />
                            <p class="description">
                                <?php printf(
                                    __('Personal Access Token with read access. For better security, define %s in wp-config.php instead.', '{{TEXT_DOMAIN}}'),
                                    '<code>{{PREFIX_UPPER}}_GITHUB_TOKEN</code>'
                                ); ?>
                            </p>
                        </td>
                    </tr>
                    <?php endif; ?>
                </tbody>
            </table>

            <?php submit_button(); ?>
        </form>

        <hr />

        <h2><?php _e('Status', '{{TEXT_DOMAIN}}'); ?></h2>

        <form method="post">
            <?php wp_nonce_field('{{PREFIX}}_updater_settings-options'); ?>

            <table class="form-table">
                <tbody>
                    <tr>
                        <th scope="row"><?php _e('Status', '{{TEXT_DOMAIN}}'); ?></th>
                        <td>
                            <?php if ($is_configured): ?>
                                <span class="dashicons dashicons-yes-alt" style="color: green;"></span>
                                <?php _e('Configured and active', '{{TEXT_DOMAIN}}'); ?>
                            <?php else: ?>
                                <span class="dashicons dashicons-warning" style="color: orange;"></span>
                                <?php _e('GitHub token not configured', '{{TEXT_DOMAIN}}'); ?>
                            <?php endif; ?>
                        </td>
                    </tr>
                    <tr>
                        <th scope="row"><?php _e('Current Version', '{{TEXT_DOMAIN}}'); ?></th>
                        <td><code><?php echo esc_html($current_version); ?></code></td>
                    </tr>
                    <tr>
                        <th scope="row"><?php _e('Latest Version', '{{TEXT_DOMAIN}}'); ?></th>
                        <td>
                            <code><?php echo esc_html($latest_version); ?></code>
                            <?php if (!is_wp_error($release) && version_compare($latest_version, $current_version, '>')): ?>
                                <span style="color: green; font-weight: bold;">
                                    &mdash; <?php _e('Update available!', '{{TEXT_DOMAIN}}'); ?>
                                </span>
                            <?php endif; ?>
                        </td>
                    </tr>
                    <?php if ($last_check): ?>
                    <tr>
                        <th scope="row"><?php _e('Last Checked', '{{TEXT_DOMAIN}}'); ?></th>
                        <td><?php echo esc_html($last_check); ?></td>
                    </tr>
                    <?php endif; ?>
                    <?php if ($is_configured): ?>
                    <tr>
                        <th scope="row"><?php _e('Check for Updates', '{{TEXT_DOMAIN}}'); ?></th>
                        <td>
                            <button type="submit" name="{{PREFIX}}_check_now" class="button">
                                <?php _e('Check Now', '{{TEXT_DOMAIN}}'); ?>
                            </button>
                            <p class="description"><?php _e('Force check (clears cache).', '{{TEXT_DOMAIN}}'); ?></p>
                        </td>
                    </tr>
                    <?php endif; ?>
                </tbody>
            </table>
        </form>
    </div>
    <?php
}
```

---

## File 6: admin-notice.php (PRIVATE REPOS ONLY)

Skip this file entirely for public repositories.

```php
<?php
/**
 * GitHub Theme Updater Admin Notices
 *
 * Guides the admin to configure the GitHub token when it's missing.
 */

if (!defined('ABSPATH')) {
    exit;
}

function {{PREFIX}}_updater_admin_notice() {
    if (!current_user_can('manage_options')) {
        return;
    }

    // Don't show on the settings page itself
    if (isset($_GET['page']) && $_GET['page'] === '{{PREFIX}}-github-updater') {
        return;
    }

    $client = {{PREFIX}}_github_client();
    if ($client->is_configured()) {
        return;
    }

    // Check if user dismissed this notice
    if (get_user_meta(get_current_user_id(), '{{PREFIX}}_updater_notice_dismissed', true)) {
        return;
    }

    // Only show on relevant admin pages
    $screen = get_current_screen();
    if (!in_array($screen->id, array('themes', 'update-core', 'dashboard'))) {
        return;
    }

    $settings_url = admin_url('themes.php?page={{PREFIX}}-github-updater');
    ?>
    <div class="notice notice-warning is-dismissible" id="{{PREFIX}}-updater-notice">
        <p>
            <strong><?php _e('{{THEME_NAME}} Updates:', '{{TEXT_DOMAIN}}'); ?></strong>
            <?php _e('GitHub token not configured. Automatic theme updates are disabled.', '{{TEXT_DOMAIN}}'); ?>
        </p>
        <p>
            <?php _e('Add to wp-config.php:', '{{TEXT_DOMAIN}}'); ?>
            <code>define('{{PREFIX_UPPER}}_GITHUB_TOKEN', 'your-token-here');</code>
        </p>
        <p>
            <a href="<?php echo esc_url($settings_url); ?>" class="button button-primary">
                <?php _e('Configure Settings', '{{TEXT_DOMAIN}}'); ?>
            </a>
            <a href="https://github.com/settings/tokens" target="_blank" class="button">
                <?php _e('Generate Token', '{{TEXT_DOMAIN}}'); ?>
            </a>
        </p>
    </div>
    <script>
    jQuery(document).ready(function($) {
        $('#{{PREFIX}}-updater-notice').on('click', '.notice-dismiss', function() {
            $.post(ajaxurl, {
                action: '{{PREFIX}}_dismiss_updater_notice',
                nonce: '<?php echo wp_create_nonce('{{PREFIX}}_dismiss_updater_notice'); ?>'
            });
        });
    });
    </script>
    <?php
}
add_action('admin_notices', '{{PREFIX}}_updater_admin_notice');

function {{PREFIX}}_dismiss_updater_notice() {
    check_ajax_referer('{{PREFIX}}_dismiss_updater_notice', 'nonce');
    if (!current_user_can('manage_options')) {
        wp_die();
    }
    update_user_meta(get_current_user_id(), '{{PREFIX}}_updater_notice_dismissed', 1);
    wp_die();
}
add_action('wp_ajax_{{PREFIX}}_dismiss_updater_notice', '{{PREFIX}}_dismiss_updater_notice');
```

---

## Integration with functions.php

After generating the updater files, ensure the loader is included in the theme's `functions.php`. If the theme already auto-loads files from `includes/`, no change is needed. Otherwise, add near the top of `functions.php`:

```php
// GitHub Theme Updater
require_once get_stylesheet_directory() . '/includes/github-updater/loader.php';
```

For child themes, `get_stylesheet_directory()` returns the child theme path. For standalone themes, it returns the theme path. Both are correct.
