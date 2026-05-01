# WordPress Snippets DE 🇨🇭

**Bewährte PHP-Snippets für professionelle WordPress-Projekte im DACH-Raum**

Zusammengestellt von [web-echo.ch](https://web-echo.ch) — SEO- und WordPress-Agentur aus der Schweiz.  
Alle Snippets sind in aktiven Kundenprojekten getestet (WordPress 6.x, PHP 8.x).

---

## Inhalt

- [Performance](#performance)
- [SEO & Schema](#seo--schema)
- [Admin-Verbesserungen](#admin-verbesserungen)
- [Sicherheit](#sicherheit)
- [Nützliches für Agenturen](#nützliches-für-agenturen)

---

## Verwendung

Alle Snippets gehören in die `functions.php` des Child-Themes oder in ein eigenes Plugin (empfohlen: [Code Snippets Plugin](https://wordpress.org/plugins/code-snippets/)).

**Niemals** direkt in die `functions.php` des Parent-Themes einfügen — geht beim nächsten Theme-Update verloren.

---

## Performance

### Emojis deaktivieren (spart ~10 KB pro Seitenaufruf)

```php
add_action('init', function() {
    remove_action('wp_head', 'print_emoji_detection_script', 7);
    remove_action('admin_print_scripts', 'print_emoji_detection_script');
    remove_action('wp_print_styles', 'print_emoji_styles');
    remove_action('admin_print_styles', 'print_emoji_styles');
    remove_filter('the_content_feed', 'wp_staticize_emoji');
    remove_filter('comment_text_rss', 'wp_staticize_emoji');
    remove_filter('wp_mail', 'wp_staticize_emoji_for_email');
});
```

**Wann verwenden:** Immer, ausser der Kunde benötigt Emoji in Beiträgen explizit.

---

### jQuery aus dem Footer laden

```php
add_action('wp_enqueue_scripts', function() {
    if (!is_admin()) {
        wp_deregister_script('jquery');
        wp_register_script('jquery', includes_url('/js/jquery/jquery.min.js'), false, null, true);
        wp_enqueue_script('jquery');
    }
});
```

**Wann verwenden:** Bei Themes, die jQuery für Interaktionen im sichtbaren Bereich nicht brauchen. Testen vor Live-Schaltung.

---

### Heartbeat API einschränken (entlastet Server)

```php
add_filter('heartbeat_settings', function($settings) {
    $settings['interval'] = 60; // Standard: 15 Sek., hier: 60 Sek.
    return $settings;
});

// Im Frontend komplett deaktivieren
add_action('init', function() {
    if (!is_admin()) {
        wp_deregister_script('heartbeat');
    }
});
```

---

### Revisions begrenzen

```php
// In wp-config.php (vor dem WordPress-Include)
define('WP_POST_REVISIONS', 5);
```

**Hinweis:** Alternativ in `functions.php`:

```php
add_filter('wp_revisions_to_keep', function($num, $post) {
    return 5;
}, 10, 2);
```

---

## SEO & Schema

### Yoast SEO: Breadcrumb-Schema für Custom Post Types aktivieren

```php
add_filter('wpseo_breadcrumb_links', function($links) {
    // Eigene CPT-Seite als Zwischenschritt hinzufügen
    return $links;
});
```

---

### LocalBusiness Schema für Schweizer KMU (ohne Plugin)

```php
add_action('wp_head', function() {
    if (!is_front_page()) return;
    
    $schema = [
        '@context' => 'https://schema.org',
        '@type'    => 'LocalBusiness',
        'name'     => get_bloginfo('name'),
        'url'      => home_url(),
        'logo'     => [
            '@type' => 'ImageObject',
            'url'   => get_site_icon_url(512),
        ],
        'address' => [
            '@type'           => 'PostalAddress',
            'addressCountry'  => 'CH',
            'addressLocality' => 'Bern', // anpassen
        ],
        'areaServed' => [
            ['@type' => 'Country', 'name' => 'Schweiz'],
        ],
        'sameAs' => [
            'https://www.linkedin.com/company/DEIN-UNTERNEHMEN',
            'https://www.wikidata.org/wiki/DEIN-WIKIDATA-ITEM',
        ],
    ];
    
    echo '<script type="application/ld+json">' . json_encode($schema, JSON_UNESCAPED_UNICODE | JSON_PRETTY_PRINT) . '</script>';
});
```

**Wichtig für Schweizer Unternehmen:** `areaServed` explizit auf Schweiz setzen, auch wenn keine physische Adresse vorhanden — verbessert Local-SEO-Signale ohne NAP-Pflicht.

---

### Canonical URL korrigieren bei paginierten Archiven

```php
add_filter('get_canonical_url', function($canonical_url, $post) {
    if (is_paged()) {
        $canonical_url = get_pagenum_link(get_query_var('paged'));
    }
    return $canonical_url;
}, 10, 2);
```

---

### XML-Sitemap: Autoren-Seiten ausschliessen (bei Single-Author-Sites)

```php
// Für Yoast SEO
add_filter('wpseo_sitemap_exclude_author', '__return_true');

// Für RankMath
add_filter('rank_math/sitemap/exclude_post_type', function($excluded, $post_type) {
    if ($post_type === 'author') {
        return true;
    }
    return $excluded;
}, 10, 2);
```

---

## Admin-Verbesserungen

### Dashboard-Widgets entfernen (sauberere Übergabe an Kunden)

```php
add_action('wp_dashboard_setup', function() {
    remove_meta_box('dashboard_primary',       'dashboard', 'side'); // WordPress News
    remove_meta_box('dashboard_quick_press',   'dashboard', 'side'); // Schnell-Entwurf
    remove_meta_box('dashboard_right_now',     'dashboard', 'normal'); // Auf einen Blick
    remove_meta_box('dashboard_activity',      'dashboard', 'normal'); // Aktivität
});
```

---

### Eigenes Login-Logo (für White-Label-Übergaben)

```php
add_action('login_enqueue_scripts', function() {
    $logo_url = get_stylesheet_directory_uri() . '/img/logo.svg';
    echo "<style>
        #login h1 a {
            background-image: url('{$logo_url}');
            background-size: contain;
            width: 200px;
            height: 80px;
        }
    </style>";
});

add_filter('login_headerurl', fn() => home_url());
add_filter('login_headertext', fn() => get_bloginfo('name'));
```

---

### Admin-Fusszeile anpassen

```php
add_filter('admin_footer_text', fn() => 'Entwickelt von <a href="https://web-echo.ch" target="_blank">web-echo.ch</a>');
remove_filter('update_footer', 'core_update_footer');
```

---

## Sicherheit

### XML-RPC deaktivieren

```php
add_filter('xmlrpc_enabled', '__return_false');
```

**Wann:** Immer, ausser ihr nutzt JetPack mit WordPress.com-Sync.

---

### Dateibearbeitung im Admin deaktivieren

```php
// In wp-config.php
define('DISALLOW_FILE_EDIT', true);
define('DISALLOW_FILE_MODS', true); // verhindert auch Plugin-/Theme-Installs via Admin
```

---

### Autor-Enumeration blockieren

```php
add_action('template_redirect', function() {
    if (is_author() && !current_user_can('edit_posts')) {
        wp_redirect(home_url(), 301);
        exit;
    }
});

// Zusätzlich: ?author=1 redirect blockieren
add_action('init', function() {
    if (preg_match('/author=([0-9]*)/i', $_SERVER['QUERY_STRING'])) {
        wp_redirect(home_url(), 301);
        exit;
    }
});
```

---

### REST-API für nicht eingeloggte User einschränken

```php
add_filter('rest_authentication_errors', function($result) {
    if (!empty($result)) return $result;
    if (!is_user_logged_in()) {
        return new WP_Error('rest_not_logged_in', 'Kein Zugriff.', ['status' => 401]);
    }
    return $result;
});
```

**Hinweis:** Nur einsetzen, wenn keine öffentliche API benötigt wird (z.B. kein Headless-WP, kein öffentlicher Block-Editor-Zugriff).

---

## Nützliches für Agenturen

### Wartungsmodus mit eigenem Design

```php
add_action('get_header', function() {
    if (get_option('mein_wartungsmodus') !== '1') return;
    if (current_user_can('edit_posts')) return; // Admins sehen die Site normal
    
    http_response_code(503);
    header('Retry-After: 3600');
    
    echo '<!DOCTYPE html><html lang="de"><head>
        <meta charset="UTF-8">
        <title>Wartung — ' . get_bloginfo('name') . '</title>
        <meta name="robots" content="noindex">
        <style>body{font-family:sans-serif;text-align:center;padding:10vh 2rem;}</style>
    </head><body>
        <h1>Kurz in der Wartung</h1>
        <p>Wir sind gleich zurück.</p>
    </body></html>';
    exit;
});
```

Aktivieren: `update_option('mein_wartungsmodus', '1');`  
Deaktivieren: `update_option('mein_wartungsmodus', '0');`

---

### Letztes Bearbeitungsdatum im Admin sichtbar machen

```php
add_filter('manage_posts_columns', function($columns) {
    $columns['last_modified'] = 'Zuletzt bearbeitet';
    return $columns;
});

add_action('manage_posts_custom_column', function($column, $post_id) {
    if ($column === 'last_modified') {
        echo get_the_modified_date('d.m.Y H:i', $post_id);
    }
}, 10, 2);

add_filter('manage_edit-post_sortable_columns', function($columns) {
    $columns['last_modified'] = 'modified';
    return $columns;
});
```

---

## Lizenz

MIT — frei nutzbar, auch in Kundenprojekten. Kein Attributions-Pflicht.

---

## Maintainer

[web-echo.ch](https://web-echo.ch) — SEO- & WordPress-Agentur, Schweiz  
Fehler gefunden? Pull Request oder [Issue](https://github.com/simple-online-marketing/wordpress-snippets-de/issues) öffnen.
