<?php
/**
 * Système de Liste d'Attente pour WooCommerce
 * Version Sécurisée & Optimisée - EMAILS ADMIN CORRIGÉS
 * À ajouter via WP Code (PHP Snippet)
 */

// =====================================================
// 1. CRÉATION TABLE (Sécurisée + Index optimisés)
// =====================================================
function waitlist_create_table() {
    global $wpdb;
    $table_name = $wpdb->prefix . 'product_waitlist';
    $charset_collate = $wpdb->get_charset_collate();

    $sql = "CREATE TABLE IF NOT EXISTS $table_name (
        id mediumint(9) NOT NULL AUTO_INCREMENT,
        product_id bigint(20) NOT NULL,
        customer_email varchar(100) NOT NULL,
        customer_name varchar(100) DEFAULT NULL,
        date_requested datetime DEFAULT CURRENT_TIMESTAMP,
        notified tinyint(1) DEFAULT 0,
        PRIMARY KEY  (id),
        KEY product_id (product_id),
        KEY customer_email (customer_email),
        KEY notified (notified)
    ) $charset_collate;";

    require_once(ABSPATH . 'wp-admin/includes/upgrade.php');
    dbDelta($sql);
}
add_action('init', 'waitlist_create_table');

// =====================================================
// 2. RATE LIMITING (Protection anti-spam)
// =====================================================
function waitlist_check_rate_limit($email) {
    $transient_key = 'waitlist_limit_' . md5($email);
    $attempts = get_transient($transient_key);
    
    if ($attempts && $attempts >= 5) {
        return false; // Max 5 inscriptions par heure
    }
    
    set_transient($transient_key, ($attempts ? $attempts + 1 : 1), HOUR_IN_SECONDS);
    return true;
}

// =====================================================
// 3. BOUTON FRONTEND (Identique à l'original)
// =====================================================
function waitlist_add_button() {
    global $product;
    
    // Sécurité : vérifier que $product existe
    if (!$product || !is_object($product) || !method_exists($product, 'is_in_stock')) {
        return;
    }
    
    if (!$product->is_in_stock()) {
        ?>
        <div class="waitlist-container" style="margin: 20px 0;">
            <button type="button" class="waitlist-trigger-btn" id="waitlist-open-form">
                <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                    <path d="M19 21l-7-5-7 5V5a2 2 0 0 1 2-2h10a2 2 0 0 1 2 2z"></path>
                </svg>
                Me notifier quand disponible
            </button>
            
            <div id="waitlist-form-modal" class="waitlist-modal" style="display:none;">
                <div class="waitlist-modal-content">
                    <span class="waitlist-close">&times;</span>
                    <h3>🔔 Recevoir une notification</h3>
                    <p>Entrez votre email pour être informé dès que ce produit sera de nouveau en stock.</p>
                    
                    <form id="waitlist-form" class="waitlist-form">
                        <input type="hidden" name="product_id" value="<?php echo esc_attr($product->get_id()); ?>">
                        <input type="hidden" name="action" value="waitlist_subscribe">
                        <?php wp_nonce_field('waitlist_nonce', 'waitlist_nonce'); ?>
                        
                        <div class="waitlist-field">
                            <label>Nom (optionnel)</label>
                            <input type="text" name="customer_name" placeholder="Votre nom" maxlength="100">
                        </div>
                        
                        <div class="waitlist-field">
                            <label>Email *</label>
                            <input type="email" name="customer_email" required placeholder="votre@email.com" maxlength="100">
                        </div>
                        
                        <!-- Honeypot anti-bot (invisible) -->
                        <input type="text" name="website" style="display:none;" tabindex="-1" autocomplete="off">
                        
                        <button type="submit" class="waitlist-submit-btn">
                            S'inscrire à la liste d'attente
                        </button>
                        
                        <div class="waitlist-message" style="display:none;"></div>
                    </form>
                </div>
            </div>
        </div>

        <style>
            .waitlist-trigger-btn {
                display: inline-flex;
                align-items: center;
                gap: 8px;
                background: #0746C0;
                color: white;
                border: none;
                padding: 14px 28px;
                font-size: 16px;
                font-weight: 600;
                border-radius: 8px;
                cursor: pointer;
                transition: all 0.3s ease;
                box-shadow: 0 4px 15px rgba(7, 70, 192, 0.3);
                width: 100%;
                justify-content: center;
            }
            
            .waitlist-trigger-btn:hover {
                transform: translateY(-2px);
                background: #05389a;
                box-shadow: 0 6px 20px rgba(7, 70, 192, 0.4);
            }
            
            .waitlist-modal {
                display: none;
                position: fixed;
                z-index: 999999;
                left: 0;
                top: 0;
                width: 100%;
                height: 100%;
                background-color: rgba(0,0,0,0.6);
                backdrop-filter: blur(5px);
                animation: fadeIn 0.3s ease;
            }
            
            @keyframes fadeIn {
                from { opacity: 0; }
                to { opacity: 1; }
            }
            
            .waitlist-modal-content {
                background: white;
                margin: 5% auto;
                padding: 40px;
                border-radius: 16px;
                width: 90%;
                max-width: 500px;
                position: relative;
                box-shadow: 0 20px 60px rgba(0,0,0,0.3);
                animation: slideUp 0.4s ease;
            }
            
            @keyframes slideUp {
                from { 
                    transform: translateY(50px);
                    opacity: 0;
                }
                to { 
                    transform: translateY(0);
                    opacity: 1;
                }
            }
            
            .waitlist-close {
                position: absolute;
                right: 20px;
                top: 20px;
                font-size: 32px;
                font-weight: bold;
                color: #aaa;
                cursor: pointer;
                transition: color 0.3s;
            }
            
            .waitlist-close:hover {
                color: #000;
            }
            
            .waitlist-modal-content h3 {
                margin: 0 0 10px 0;
                font-size: 24px;
                color: #333;
            }
            
            .waitlist-modal-content p {
                color: #666;
                margin-bottom: 25px;
                line-height: 1.6;
            }
            
            .waitlist-field {
                margin-bottom: 20px;
            }
            
            .waitlist-field label {
                display: block;
                margin-bottom: 8px;
                font-weight: 600;
                color: #333;
                font-size: 14px;
            }
            
            .waitlist-field input {
                width: 100%;
                padding: 14px;
                border: 2px solid #e0e0e0;
                border-radius: 8px;
                font-size: 15px;
                transition: border-color 0.3s;
                box-sizing: border-box;
            }
            
            .waitlist-field input:focus {
                outline: none;
                border-color: #0746C0;
            }
            
            .waitlist-submit-btn {
                width: 100%;
                background: #0746C0;
                color: white;
                border: none;
                padding: 16px;
                font-size: 16px;
                font-weight: 600;
                border-radius: 8px;
                cursor: pointer;
                transition: all 0.3s ease;
                margin-top: 10px;
            }
            
            .waitlist-submit-btn:hover {
                transform: translateY(-2px);
                background: #05389a;
                box-shadow: 0 6px 20px rgba(7, 70, 192, 0.4);
            }
            
            .waitlist-submit-btn:disabled {
                opacity: 0.6;
                cursor: not-allowed;
            }
            
            .waitlist-message {
                margin-top: 20px;
                padding: 15px;
                border-radius: 8px;
                font-weight: 500;
            }
            
            .waitlist-message.success {
                background: #d4edda;
                color: #155724;
                border: 1px solid #c3e6cb;
            }
            
            .waitlist-message.error {
                background: #f8d7da;
                color: #721c24;
                border: 1px solid #f5c6cb;
            }
            
            @media (max-width: 768px) {
                .waitlist-modal-content {
                    width: 95%;
                    margin: 10% auto;
                    padding: 30px 20px;
                }
                
                .waitlist-trigger-btn {
                    font-size: 14px;
                    padding: 12px 20px;
                }
            }
        </style>

        <script>
        jQuery(document).ready(function($) {
            $('#waitlist-open-form').on('click', function() {
                $('#waitlist-form-modal').fadeIn(300);
            });
            
            $('.waitlist-close, .waitlist-modal').on('click', function(e) {
                if (e.target === this) {
                    $('#waitlist-form-modal').fadeOut(300);
                }
            });
            
            $('#waitlist-form').on('submit', function(e) {
                e.preventDefault();
                
                var $form = $(this);
                var $btn = $form.find('.waitlist-submit-btn');
                var $message = $form.find('.waitlist-message');
                
                // Protection honeypot
                if ($form.find('input[name="website"]').val() !== '') {
                    return false;
                }
                
                $btn.prop('disabled', true).text('Inscription en cours...');
                $message.hide();
                
                $.ajax({
                    url: '<?php echo esc_url(admin_url('admin-ajax.php')); ?>',
                    type: 'POST',
                    data: $form.serialize(),
                    timeout: 10000,
                    success: function(response) {
                        $message.removeClass('error success');
                        
                        if (response.success) {
                            $message.addClass('success').html('✅ ' + response.data.message).fadeIn();
                            $form[0].reset();
                            
                            setTimeout(function() {
                                $('#waitlist-form-modal').fadeOut(300);
                                $message.hide();
                            }, 3000);
                        } else {
                            $message.addClass('error').html('❌ ' + response.data.message).fadeIn();
                        }
                    },
                    error: function() {
                        $message.addClass('error').html('❌ Une erreur est survenue. Réessayez.').fadeIn();
                    },
                    complete: function() {
                        $btn.prop('disabled', false).text('S\'inscrire à la liste d\'attente');
                    }
                });
            });
        });
        </script>
        <?php
    }
}
add_action('woocommerce_single_product_summary', 'waitlist_add_button', 31);

// =====================================================
// 4. TRAITEMENT AJAX (✅ EMAIL ADMIN IMMÉDIAT)
// =====================================================
function waitlist_handle_subscription() {
    // Sécurité 1: Vérification nonce
    check_ajax_referer('waitlist_nonce', 'waitlist_nonce');
    
    global $wpdb;
    $table_name = $wpdb->prefix . 'product_waitlist';
    
    // Sécurité 2: Honeypot anti-bot
    if (!empty($_POST['website'])) {
        wp_send_json_error(['message' => 'Erreur de sécurité']);
    }
    
    // Sécurité 3: Sanitization stricte
    $product_id = isset($_POST['product_id']) ? absint($_POST['product_id']) : 0;
    $customer_email = isset($_POST['customer_email']) ? sanitize_email($_POST['customer_email']) : '';
    $customer_name = isset($_POST['customer_name']) ? sanitize_text_field(wp_unslash($_POST['customer_name'])) : '';
    
    // Validation email
    if (!is_email($customer_email)) {
        wp_send_json_error(['message' => 'Email invalide']);
    }
    
    // Sécurité 4: Vérifier que le produit existe
    $product = wc_get_product($product_id);
    if (!$product) {
        wp_send_json_error(['message' => 'Produit introuvable']);
    }
    
    // Sécurité 5: Rate limiting (5 max par heure)
    if (!waitlist_check_rate_limit($customer_email)) {
        wp_send_json_error(['message' => 'Trop de tentatives. Réessayez dans 1 heure.']);
    }
    
    // Vérifier si déjà inscrit (requête préparée = protection SQL injection)
    $exists = $wpdb->get_var($wpdb->prepare(
        "SELECT id FROM $table_name WHERE product_id = %d AND customer_email = %s",
        $product_id, $customer_email
    ));
    
    if ($exists) {
        wp_send_json_error(['message' => 'Vous êtes déjà inscrit à cette liste d\'attente']);
    }
    
    // Insertion sécurisée (requête préparée)
    $inserted = $wpdb->insert(
        $table_name,
        [
            'product_id' => $product_id,
            'customer_email' => $customer_email,
            'customer_name' => $customer_name,
            'notified' => 0
        ],
        ['%d', '%s', '%s', '%d']
    );
    
    if ($inserted) {
        // ✅ CORRECTION : Envoi IMMÉDIAT de l'email admin (pas de schedule)
        waitlist_send_admin_email($product_id, $customer_email, $customer_name);
        
        wp_send_json_success(['message' => 'Inscription réussie ! Vous serez notifié dès que le produit sera disponible.']);
    } else {
        wp_send_json_error(['message' => 'Erreur lors de l\'inscription']);
    }
}
add_action('wp_ajax_waitlist_subscribe', 'waitlist_handle_subscription');
add_action('wp_ajax_nopriv_waitlist_subscribe', 'waitlist_handle_subscription');

// =====================================================
// 5. EMAIL ADMIN (✅ Appel direct, pas de hook)
// =====================================================
function waitlist_send_admin_email($product_id, $customer_email, $customer_name) {
    $product = wc_get_product($product_id);
    if (!$product) return;
    
    $admin_email = get_option('admin_email');
    $product_link = admin_url('post.php?post=' . $product_id . '&action=edit');
    
    $admin_html = '<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
</head>
<body style="margin:0;padding:0;font-family:-apple-system,BlinkMacSystemFont,\'Segoe UI\',Roboto,\'Helvetica Neue\',Arial,sans-serif;background:#f4f6f9;">
    <table role="presentation" width="100%" cellpadding="0" cellspacing="0" border="0" style="background:#f4f6f9;padding:40px 20px;">
        <tr>
            <td align="center">
                <table role="presentation" width="100%" cellpadding="0" cellspacing="0" border="0" style="max-width:600px;background:#ffffff;border-radius:12px;overflow:hidden;box-shadow:0 4px 20px rgba(0,0,0,0.08);">
                    
                    <!-- Header -->
                    <tr>
                        <td style="background:linear-gradient(135deg,#0746C0 0%,#0532a0 100%);padding:30px 25px;text-align:center;">
                            <h1 style="margin:0;color:#ffffff;font-size:24px;font-weight:700;line-height:1.3;">
                                🔔 Nouvelle inscription<br>
                                <span style="font-size:16px;font-weight:500;opacity:0.9;">Liste d\'attente</span>
                            </h1>
                        </td>
                    </tr>
                    
                    <!-- Content -->
                    <tr>
                        <td style="padding:35px 25px;">
                            
                            <!-- Alert Box -->
                            <table role="presentation" width="100%" cellpadding="0" cellspacing="0" border="0" style="margin-bottom:25px;background:#fff3cd;border-left:4px solid #0746C0;border-radius:8px;">
                                <tr>
                                    <td style="padding:15px 20px;">
                                        <p style="margin:0;color:#856404;font-size:14px;font-weight:600;line-height:1.5;">
                                            ⚠️ Un client souhaite être notifié du retour en stock
                                        </p>
                                    </td>
                                </tr>
                            </table>
                            
                            <!-- Product Info -->
                            <h3 style="margin:0 0 12px 0;color:#0746C0;font-size:16px;font-weight:700;">📦 Produit concerné</h3>
                            <p style="margin:0 0 25px 0;color:#111827;font-size:18px;font-weight:700;line-height:1.4;">
                                ' . esc_html($product->get_name()) . '
                            </p>
                            
                            <!-- Client Info Table -->
                            <h3 style="margin:0 0 12px 0;color:#0746C0;font-size:16px;font-weight:700;">👤 Informations client</h3>
                            <table role="presentation" width="100%" cellpadding="12" cellspacing="0" border="0" style="background:#f8f9fa;border-radius:8px;margin-bottom:25px;">
                                <tr>
                                    <td style="width:35%;color:#6b7280;font-size:14px;font-weight:600;vertical-align:top;border-bottom:1px solid #e5e7eb;">Nom :</td>
                                    <td style="color:#111827;font-size:14px;font-weight:500;border-bottom:1px solid #e5e7eb;">
                                        ' . esc_html($customer_name ?: 'Non renseigné') . '
                                    </td>
                                </tr>
                                <tr>
                                    <td style="color:#6b7280;font-size:14px;font-weight:600;vertical-align:top;">Email :</td>
                                    <td style="color:#111827;font-size:14px;font-weight:500;word-break:break-word;">
                                        ' . esc_html($customer_email) . '
                                    </td>
                                </tr>
                            </table>
                            
                            <!-- Action Buttons -->
                            <table role="presentation" width="100%" cellpadding="0" cellspacing="0" border="0">
                                <tr>
                                    <td align="center" style="padding:10px 0;">
                                        <a href="' . esc_url($product_link) . '" 
                                           style="display:inline-block;padding:14px 28px;background:#0746C0;color:#ffffff;text-decoration:none;font-size:15px;font-weight:700;border-radius:8px;box-shadow:0 4px 12px rgba(7,70,192,0.3);">
                                            Gérer le produit →
                                        </a>
                                    </td>
                                </tr>
                            </table>
                            
                        </td>
                    </tr>
                    
                    <!-- Footer -->
                    <tr>
                        <td style="background:#f8f9fa;padding:20px 25px;text-align:center;border-top:1px solid #e5e7eb;">
                            <p style="margin:0;color:#6b7280;font-size:12px;line-height:1.5;">
                                Notification automatique - ' . esc_html(get_bloginfo('name')) . '<br>
                                Le client sera automatiquement notifié dès la remise en stock
                            </p>
                        </td>
                    </tr>
                    
                </table>
            </td>
        </tr>
    </table>
</body>
</html>';
    
    $headers = ['Content-Type: text/html; charset=UTF-8'];
    wp_mail($admin_email, '🔔 Nouvelle demande - ' . $product->get_name(), $admin_html, $headers);
}

// =====================================================
// 6. NOTIFICATIONS CLIENTS (✅ Récap admin immédiat)
// =====================================================
function waitlist_notify_customers($product_id) {
    global $wpdb;
    $table_name = $wpdb->prefix . 'product_waitlist';
    
    $product = wc_get_product($product_id);
    
    if ($product && $product->is_in_stock() && $product->get_stock_quantity() > 0) {
        // Cache pour éviter doublons
        $cache_key = 'waitlist_sent_' . $product_id;
        if (get_transient($cache_key)) {
            return; // Déjà traité récemment
        }
        set_transient($cache_key, true, 5 * MINUTE_IN_SECONDS);
        
        // Récupérer clients (requête préparée)
        $customers = $wpdb->get_results($wpdb->prepare(
            "SELECT * FROM $table_name WHERE product_id = %d AND notified = 0",
            $product_id
        ));
        
        if (!empty($customers)) {
            $product_image = wp_get_attachment_image_src(get_post_thumbnail_id($product_id), 'medium');
            $image_url = $product_image ? $product_image[0] : wc_placeholder_img_src();
            
            $price = $product->get_price();
            $currency_symbol = get_woocommerce_currency_symbol();
            $price_formatted = $currency_symbol . number_format($price, 2);
            
            foreach ($customers as $customer) {
                $to = $customer->customer_email;
                $subject = '🎉 ' . $product->get_name() . ' est de nouveau disponible !';
                
                $message = '<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
</head>
<body style="margin:0;padding:0;font-family:-apple-system,BlinkMacSystemFont,\'Segoe UI\',Roboto,\'Helvetica Neue\',Arial,sans-serif;background:#f4f6f9;">
    <table role="presentation" width="100%" cellpadding="0" cellspacing="0" border="0" style="background:#f4f6f9;padding:40px 15px;">
        <tr>
            <td align="center">
                <table role="presentation" width="100%" cellpadding="0" cellspacing="0" border="0" style="max-width:600px;background:#ffffff;border-radius:16px;overflow:hidden;box-shadow:0 8px 24px rgba(0,0,0,0.12);">
                    
                    <!-- Header -->
                    <tr>
                        <td style="background:linear-gradient(135deg,#0746C0 0%,#0532a0 100%);padding:40px 30px;text-align:center;">
                            <h1 style="margin:0;color:#ffffff;font-size:32px;font-weight:800;line-height:1.2;">
                                🎉 Bonne nouvelle !
                            </h1>
                            <p style="margin:12px 0 0 0;color:rgba(255,255,255,0.9);font-size:16px;font-weight:500;line-height:1.4;">
                                Le produit que vous attendiez est de retour
                            </p>
                        </td>
                    </tr>
                    
                    <!-- Greeting -->
                    <tr>
                        <td style="padding:35px 30px 20px 30px;">
                            <p style="margin:0 0 10px 0;color:#111827;font-size:17px;font-weight:600;line-height:1.5;">
                                Bonjour ' . esc_html($customer->customer_name ?: 'cher client') . ',
                            </p>
                            <p style="margin:0;color:#6b7280;font-size:15px;line-height:1.6;">
                                Nous sommes ravis de vous informer que le produit est maintenant <strong style="color:#10b981;">disponible en stock</strong> !
                            </p>
                        </td>
                    </tr>
                    
                    <!-- Product Card -->
                    <tr>
                        <td style="padding:0 30px 30px 30px;">
                            <table role="presentation" width="100%" cellpadding="0" cellspacing="0" border="0" style="background:linear-gradient(to bottom,#f9fafb,#f3f4f6);border-radius:12px;overflow:hidden;">
                                <tr>
                                    <td style="padding:30px 25px;text-align:center;">
                                        
                                        <!-- Product Image -->
                                        <img src="' . esc_url($image_url) . '" 
                                             alt="' . esc_attr($product->get_name()) . '" 
                                             style="max-width:200px;width:100%;height:auto;border-radius:10px;margin-bottom:20px;display:block;margin-left:auto;margin-right:auto;">
                                        
                                        <!-- Product Name -->
                                        <h2 style="margin:0 0 18px 0;color:#0746C0;font-size:22px;font-weight:800;line-height:1.3;">
                                            ' . esc_html($product->get_name()) . '
                                        </h2>
                                        
                                        <!-- Price Badge -->
                                        <div style="display:inline-block;padding:14px 32px;background:#0746C0;color:#ffffff;border-radius:30px;margin:0 0 18px 0;">
                                            <span style="font-size:32px;font-weight:800;line-height:1;">' . esc_html($price_formatted) . '</span>
                                        </div>
                                        
                                        <!-- Stock Info -->
                                        <p style="margin:0;color:#10b981;font-size:15px;font-weight:700;">
                                            ✅ En stock : ' . intval($product->get_stock_quantity()) . ' unité(s) disponible(s)
                                        </p>
                                        
                                    </td>
                                </tr>
                            </table>
                        </td>
                    </tr>
                    
                    <!-- CTA Button -->
                    <tr>
                        <td style="padding:0 30px 35px 30px;text-align:center;">
                            <a href="' . esc_url(get_permalink($product_id)) . '" 
                               style="display:inline-block;padding:18px 50px;background:linear-gradient(135deg,#0746C0 0%,#0532a0 100%);color:#ffffff;text-decoration:none;font-size:18px;font-weight:800;border-radius:30px;box-shadow:0 6px 20px rgba(7,70,192,0.4);text-align:center;">
                                🛒 Commander maintenant
                            </a>
                            <p style="margin:18px 0 0 0;color:#ef4444;font-size:14px;font-weight:700;">
                                ⚡ Dépêchez-vous ! Stock limité
                            </p>
                        </td>
                    </tr>
                    
                    <!-- Footer -->
                    <tr>
                        <td style="background:#f8f9fa;padding:30px;border-top:2px solid #e5e7eb;">
                            <p style="margin:0 0 12px 0;color:#4b5563;font-size:14px;font-weight:600;line-height:1.5;">
                                Merci de votre patience et de votre fidélité !
                            </p>
                            <p style="margin:0;color:#6b7280;font-size:13px;line-height:1.5;">
                                Cordialement,<br>
                                <strong style="color:#0746C0;">' . esc_html(get_bloginfo('name')) . '</strong>
                            </p>
                            <p style="margin:20px 0 0 0;padding-top:20px;border-top:1px solid #e5e7eb;color:#9ca3af;font-size:12px;line-height:1.5;">
                                Vous recevez cet email car vous vous êtes inscrit à notre liste d\'attente.<br>
                                Cet email a été envoyé automatiquement, merci de ne pas y répondre.
                            </p>
                        </td>
                    </tr>
                    
                </table>
            </td>
        </tr>
    </table>
</body>
</html>';
                
                $headers = ['Content-Type: text/html; charset=UTF-8', 'From: ' . get_bloginfo('name') . ' <' . get_option('admin_email') . '>'];
                
                $sent = wp_mail($to, $subject, $message, $headers);
                
                if ($sent) {
                    $wpdb->update($table_name, ['notified' => 1], ['id' => $customer->id], ['%d'], ['%d']);
                }
                
                usleep(50000); // Pause 0.05s entre envois
            }
            
            // ✅ CORRECTION : Envoi IMMÉDIAT du récap admin (pas de schedule)
            waitlist_send_admin_recap($product_id, count($customers));
        }
    }
}
add_action('woocommerce_product_set_stock', 'waitlist_notify_customers');
add_action('woocommerce_variation_set_stock', 'waitlist_notify_customers');
add_action('woocommerce_update_product', 'waitlist_notify_customers');

// =====================================================
// 7. RÉCAP ADMIN (✅ Appel direct, pas de hook)
// =====================================================
function waitlist_send_admin_recap($product_id, $count) {
    $product = wc_get_product($product_id);
    if (!$product) return;
    
    $msg = '<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
</head>
<body style="margin:0;padding:0;font-family:-apple-system,BlinkMacSystemFont,\'Segoe UI\',Roboto,\'Helvetica Neue\',Arial,sans-serif;background:#f4f6f9;">
    <table role="presentation" width="100%" cellpadding="0" cellspacing="0" border="0" style="background:#f4f6f9;padding:40px 20px;">
        <tr>
            <td align="center">
                <table role="presentation" width="100%" cellpadding="0" cellspacing="0" border="0" style="max-width:600px;background:#ffffff;border-radius:12px;overflow:hidden;box-shadow:0 4px 20px rgba(0,0,0,0.08);">
                    
                    <!-- Header -->
                    <tr>
                        <td style="background:linear-gradient(135deg,#10b981 0%,#059669 100%);padding:30px 25px;text-align:center;">
                            <h1 style="margin:0;color:#ffffff;font-size:24px;font-weight:700;line-height:1.3;">
                                ✅ Notifications envoyées<br>
                                <span style="font-size:16px;font-weight:500;opacity:0.9;">avec succès</span>
                            </h1>
                        </td>
                    </tr>
                    
                    <!-- Content -->
                    <tr>
                        <td style="padding:35px 25px;">
                            
                            <!-- Success Box -->
                            <table role="presentation" width="100%" cellpadding="0" cellspacing="0" border="0" style="margin-bottom:25px;background:#d1fae5;border-left:4px solid #10b981;border-radius:8px;">
                                <tr>
                                    <td style="padding:20px;">
                                        <p style="margin:0;color:#065f46;font-size:18px;font-weight:700;line-height:1.4;text-align:center;">
                                            <span style="font-size:42px;display:block;margin-bottom:8px;">' . intval($count) . '</span>
                                            client' . ($count > 1 ? 's ont' : ' a') . ' été notifié' . ($count > 1 ? 's' : '') . '
                                        </p>
                                    </td>
                                </tr>
                            </table>
                            
                            <!-- Product Info -->
                            <h3 style="margin:0 0 12px 0;color:#0746C0;font-size:16px;font-weight:700;">📦 Produit concerné</h3>
                            <p style="margin:0 0 8px 0;color:#111827;font-size:18px;font-weight:700;line-height:1.4;">
                                ' . esc_html($product->get_name()) . '
                            </p>
                            <p style="margin:0 0 25px 0;color:#6b7280;font-size:14px;line-height:1.5;">
                                Stock actuel : <strong style="color:#10b981;">' . intval($product->get_stock_quantity()) . ' unité(s)</strong>
                            </p>
                            
                            <!-- Info Box -->
                            <table role="presentation" width="100%" cellpadding="0" cellspacing="0" border="0" style="background:#eff6ff;border-radius:8px;margin-top:25px;">
                                <tr>
                                    <td style="padding:18px 20px;">
                                        <p style="margin:0;color:#1e40af;font-size:13px;line-height:1.6;">
                                            <strong>💡 Information :</strong> Tous les clients en attente ont reçu un email automatique avec le lien direct vers le produit.
                                        </p>
                                    </td>
                                </tr>
                            </table>
                            
                        </td>
                    </tr>
                    
                    <!-- Footer -->
                    <tr>
                        <td style="background:#f8f9fa;padding:20px 25px;text-align:center;border-top:1px solid #e5e7eb;">
                            <p style="margin:0;color:#6b7280;font-size:12px;line-height:1.5;">
                                Notification automatique - ' . esc_html(get_bloginfo('name')) . '
                            </p>
                        </td>
                    </tr>
                    
                </table>
            </td>
        </tr>
    </table>
</body>
</html>';
    
    $headers = ['Content-Type: text/html; charset=UTF-8'];
    wp_mail(get_option('admin_email'), '✅ ' . $count . ' notification(s) - ' . $product->get_name(), $msg, $headers);
}

// =====================================================
// 8. PAGE ADMIN MODERNE (Design Premium)
// =====================================================
function waitlist_admin_menu() {
    add_submenu_page('woocommerce', 'Listes d\'attente', 'Listes d\'attente', 'manage_woocommerce', 'waitlist-manager', 'waitlist_admin_page');
}
add_action('admin_menu', 'waitlist_admin_menu');

function waitlist_admin_page() {
    if (!current_user_can('manage_woocommerce')) {
        wp_die(esc_html__('Permissions insuffisantes'));
    }
    
    global $wpdb;
    $table_name = $wpdb->prefix . 'product_waitlist';
    
    // Gestion suppression sécurisée
    if (isset($_GET['action'], $_GET['id'], $_GET['_wpnonce']) && $_GET['action'] === 'delete') {
        if (wp_verify_nonce(sanitize_text_field(wp_unslash($_GET['_wpnonce'])), 'delete_waitlist_' . intval($_GET['id']))) {
            $wpdb->delete($table_name, ['id' => absint($_GET['id'])], ['%d']);
            echo '<div class="notice notice-success is-dismissible"><p><strong>✅ Entrée supprimée avec succès</strong></p></div>';
        }
    }
    
    // Filtres et recherche
    $status_filter = isset($_GET['status']) ? sanitize_text_field($_GET['status']) : 'all';
    $search = isset($_GET['s']) ? sanitize_text_field($_GET['s']) : '';
    
    $where = "WHERE 1=1";
    $params = [];
    
    if ($status_filter === 'pending') {
        $where .= " AND w.notified = 0";
    } elseif ($status_filter === 'notified') {
        $where .= " AND w.notified = 1";
    }
    
    if ($search) {
        $where .= " AND (p.post_title LIKE %s OR w.customer_email LIKE %s OR w.customer_name LIKE %s)";
        $like = '%' . $wpdb->esc_like($search) . '%';
        $params = [$like, $like, $like];
    }
    
    // Pagination
    $per_page = 15;
    $paged = isset($_GET['paged']) ? absint($_GET['paged']) : 1;
    $offset = ($paged - 1) * $per_page;
    
    // Total pour pagination
    $total_query = "SELECT COUNT(*) FROM $table_name w LEFT JOIN {$wpdb->posts} p ON w.product_id = p.ID $where";
    $total = $params ? $wpdb->get_var($wpdb->prepare($total_query, $params)) : $wpdb->get_var($total_query);
    
    // Requête principale
    $query = "SELECT w.*, p.post_title as product_name FROM $table_name w LEFT JOIN {$wpdb->posts} p ON w.product_id = p.ID $where ORDER BY w.date_requested DESC LIMIT %d OFFSET %d";
    $requests = $wpdb->get_results($wpdb->prepare($query, array_merge($params, [$per_page, $offset])));
    
    // Statistiques
    $stats_pending = $wpdb->get_var("SELECT COUNT(*) FROM $table_name WHERE notified = 0");
    $stats_notified = $wpdb->get_var("SELECT COUNT(*) FROM $table_name WHERE notified = 1");
    $stats_total = $wpdb->get_var("SELECT COUNT(*) FROM $table_name");
    $stats_today = $wpdb->get_var("SELECT COUNT(*) FROM $table_name WHERE DATE(date_requested) = CURDATE()");
    
    ?>
    <div class="wrap" style="margin: 20px 20px 20px 0;">
        <h1 style="display: flex; align-items: center; gap: 15px; margin-bottom: 30px;">
            <span style="background: linear-gradient(135deg, #0746C0, #0532a0); -webkit-background-clip: text; -webkit-text-fill-color: transparent; font-size: 32px; font-weight: 800;">
                📋 Listes d'Attente
            </span>
        </h1>
        
        <!-- Cartes Statistiques Modernes -->
        <div style="display: grid; grid-template-columns: repeat(auto-fit, minmax(240px, 1fr)); gap: 20px; margin-bottom: 30px;">
            
            <div style="background: linear-gradient(135deg, #0746C0 0%, #0532a0 100%); border-radius: 16px; padding: 25px; box-shadow: 0 8px 24px rgba(7, 70, 192, 0.25); position: relative; overflow: hidden;">
                <div style="position: absolute; top: -20px; right: -20px; width: 100px; height: 100px; background: rgba(255,255,255,0.1); border-radius: 50%;"></div>
                <div style="position: relative; z-index: 1;">
                    <div style="color: rgba(255,255,255,0.8); font-size: 13px; font-weight: 600; text-transform: uppercase; letter-spacing: 1px; margin-bottom: 10px;">⏳ En Attente</div>
                    <div style="color: white; font-size: 42px; font-weight: 800; line-height: 1; margin-bottom: 8px;"><?php echo intval($stats_pending); ?></div>
                    <div style="color: rgba(255,255,255,0.7); font-size: 12px;">Notifications à envoyer</div>
                </div>
            </div>
            
            <div style="background: linear-gradient(135deg, #10b981 0%, #059669 100%); border-radius: 16px; padding: 25px; box-shadow: 0 8px 24px rgba(16, 185, 129, 0.25); position: relative; overflow: hidden;">
                <div style="position: absolute; top: -20px; right: -20px; width: 100px; height: 100px; background: rgba(255,255,255,0.1); border-radius: 50%;"></div>
                <div style="position: relative; z-index: 1;">
                    <div style="color: rgba(255,255,255,0.8); font-size: 13px; font-weight: 600; text-transform: uppercase; letter-spacing: 1px; margin-bottom: 10px;">✅ Notifiés</div>
                    <div style="color: white; font-size: 42px; font-weight: 800; line-height: 1; margin-bottom: 8px;"><?php echo intval($stats_notified); ?></div>
                    <div style="color: rgba(255,255,255,0.7); font-size: 12px;">Clients informés</div>
                </div>
            </div>
            
            <div style="background: linear-gradient(135deg, #8b5cf6 0%, #7c3aed 100%); border-radius: 16px; padding: 25px; box-shadow: 0 8px 24px rgba(139, 92, 246, 0.25); position: relative; overflow: hidden;">
                <div style="position: absolute; top: -20px; right: -20px; width: 100px; height: 100px; background: rgba(255,255,255,0.1); border-radius: 50%;"></div>
                <div style="position: relative; z-index: 1;">
                    <div style="color: rgba(255,255,255,0.8); font-size: 13px; font-weight: 600; text-transform: uppercase; letter-spacing: 1px; margin-bottom: 10px;">📊 Total</div>
                    <div style="color: white; font-size: 42px; font-weight: 800; line-height: 1; margin-bottom: 8px;"><?php echo intval($stats_total); ?></div>
                    <div style="color: rgba(255,255,255,0.7); font-size: 12px;">Inscriptions totales</div>
                </div>
            </div>
            
            <div style="background: linear-gradient(135deg, #f59e0b 0%, #d97706 100%); border-radius: 16px; padding: 25px; box-shadow: 0 8px 24px rgba(245, 158, 11, 0.25); position: relative; overflow: hidden;">
                <div style="position: absolute; top: -20px; right: -20px; width: 100px; height: 100px; background: rgba(255,255,255,0.1); border-radius: 50%;"></div>
                <div style="position: relative; z-index: 1;">
                    <div style="color: rgba(255,255,255,0.8); font-size: 13px; font-weight: 600; text-transform: uppercase; letter-spacing: 1px; margin-bottom: 10px;">🔥 Aujourd'hui</div>
                    <div style="color: white; font-size: 42px; font-weight: 800; line-height: 1; margin-bottom: 8px;"><?php echo intval($stats_today); ?></div>
                    <div style="color: rgba(255,255,255,0.7); font-size: 12px;">Nouvelles demandes</div>
                </div>
            </div>
            
        </div>
        
        <!-- Panneau Principal Moderne -->
        <div style="background: white; border-radius: 16px; box-shadow: 0 4px 20px rgba(0,0,0,0.08); overflow: hidden;">
            
            <!-- Header avec Filtres -->
            <div style="padding: 25px 30px; border-bottom: 1px solid #f0f0f0;">
                <form method="get" style="display: flex; gap: 15px; align-items: center; flex-wrap: wrap;">
                    <input type="hidden" name="page" value="waitlist-manager">
                    
                    <div style="flex: 1; min-width: 250px;">
                        <div style="position: relative;">
                            <input type="search" name="s" value="<?php echo esc_attr($search); ?>" 
                                   placeholder="🔍 Rechercher un produit, email, nom..." 
                                   style="width: 100%; padding: 12px 20px 12px 45px; border: 2px solid #e5e7eb; border-radius: 10px; font-size: 14px; transition: all 0.3s;"
                                   onfocus="this.style.borderColor='#0746C0'; this.style.boxShadow='0 0 0 3px rgba(7,70,192,0.1)';"
                                   onblur="this.style.borderColor='#e5e7eb'; this.style.boxShadow='none';">
                            <span style="position: absolute; left: 16px; top: 50%; transform: translateY(-50%); color: #9ca3af; font-size: 18px;">🔍</span>
                        </div>
                    </div>
                    
                    <select name="status" 
                            style="padding: 12px 40px 12px 20px; border: 2px solid #e5e7eb; border-radius: 10px; font-size: 14px; font-weight: 600; background: white; cursor: pointer; appearance: none; background-image: url('data:image/svg+xml;charset=UTF-8,%3csvg width=\'10\' height=\'6\' viewBox=\'0 0 10 6\' fill=\'none\' xmlns=\'http://www.w3.org/2000/svg\'%3e%3cpath d=\'M1 1L5 5L9 1\' stroke=\'%239ca3af\' stroke-width=\'1.5\' stroke-linecap=\'round\' stroke-linejoin=\'round\'/%3e%3c/svg%3e'); background-repeat: no-repeat; background-position: right 15px center;">
                        <option value="all" <?php selected($status_filter, 'all'); ?>>📋 Tous les statuts</option>
                        <option value="pending" <?php selected($status_filter, 'pending'); ?>>⏳ En attente</option>
                        <option value="notified" <?php selected($status_filter, 'notified'); ?>>✅ Notifiés</option>
                    </select>
                    
                    <button type="submit" 
                            style="padding: 12px 28px; background: linear-gradient(135deg, #0746C0, #0532a0); color: white; border: none; border-radius: 10px; font-size: 14px; font-weight: 700; cursor: pointer; transition: all 0.3s; box-shadow: 0 4px 12px rgba(7,70,192,0.2);"
                            onmouseover="this.style.transform='translateY(-2px)'; this.style.boxShadow='0 6px 20px rgba(7,70,192,0.3)';"
                            onmouseout="this.style.transform='translateY(0)'; this.style.boxShadow='0 4px 12px rgba(7,70,192,0.2)';">
                        Filtrer
                    </button>
                    
                    <?php if ($search || $status_filter !== 'all') : ?>
                        <a href="<?php echo esc_url(admin_url('admin.php?page=waitlist-manager')); ?>" 
                           style="padding: 12px 24px; background: #f3f4f6; color: #4b5563; border-radius: 10px; font-size: 14px; font-weight: 600; text-decoration: none; transition: all 0.3s;"
                           onmouseover="this.style.background='#e5e7eb';"
                           onmouseout="this.style.background='#f3f4f6';">
                            ↺ Réinitialiser
                        </a>
                    <?php endif; ?>
                </form>
            </div>
            
            <!-- Tableau Moderne -->
            <div style="overflow-x: auto;">
                <table style="width: 100%; border-collapse: separate; border-spacing: 0;">
                    <thead>
                        <tr style="background: linear-gradient(to right, #f9fafb, #f3f4f6);">
                            <th style="padding: 18px 25px; text-align: left; font-size: 12px; font-weight: 700; color: #6b7280; text-transform: uppercase; letter-spacing: 0.5px; border-bottom: 2px solid #e5e7eb;">📦 Produit</th>
                            <th style="padding: 18px 25px; text-align: left; font-size: 12px; font-weight: 700; color: #6b7280; text-transform: uppercase; letter-spacing: 0.5px; border-bottom: 2px solid #e5e7eb;">👤 Client</th>
                            <th style="padding: 18px 25px; text-align: left; font-size: 12px; font-weight: 700; color: #6b7280; text-transform: uppercase; letter-spacing: 0.5px; border-bottom: 2px solid #e5e7eb;">📧 Email</th>
                            <th style="padding: 18px 25px; text-align: left; font-size: 12px; font-weight: 700; color: #6b7280; text-transform: uppercase; letter-spacing: 0.5px; border-bottom: 2px solid #e5e7eb;">📅 Date</th>
                            <th style="padding: 18px 25px; text-align: center; font-size: 12px; font-weight: 700; color: #6b7280; text-transform: uppercase; letter-spacing: 0.5px; border-bottom: 2px solid #e5e7eb;">⚡ Statut</th>
                            <th style="padding: 18px 25px; text-align: center; font-size: 12px; font-weight: 700; color: #6b7280; text-transform: uppercase; letter-spacing: 0.5px; border-bottom: 2px solid #e5e7eb;">🔧 Actions</th>
                        </tr>
                    </thead>
                    <tbody>
                        <?php if (empty($requests)) : ?>
                            <tr>
                                <td colspan="6" style="padding: 60px 25px; text-align: center;">
                                    <div style="opacity: 0.5;">
                                        <div style="font-size: 48px; margin-bottom: 15px;">📭</div>
                                        <div style="font-size: 16px; font-weight: 600; color: #6b7280; margin-bottom: 5px;">
                                            <?php echo $search ? 'Aucun résultat trouvé' : 'Aucune demande pour le moment'; ?>
                                        </div>
                                        <div style="font-size: 14px; color: #9ca3af;">
                                            <?php echo $search ? 'Essayez avec d\'autres termes de recherche' : 'Les demandes apparaîtront ici'; ?>
                                        </div>
                                    </div>
                                </td>
                            </tr>
                        <?php else : ?>
                            <?php foreach ($requests as $request) : ?>
                                <tr style="border-bottom: 1px solid #f3f4f6; transition: all 0.2s;"
                                    onmouseover="this.style.background='#f9fafb';"
                                    onmouseout="this.style.background='white';">
                                    <td style="padding: 20px 25px;">
                                        <div style="font-weight: 700; color: #111827; font-size: 14px; margin-bottom: 3px;">
                                            <?php echo esc_html($request->product_name ?: '❌ Produit supprimé'); ?>
                                        </div>
                                        <div style="font-size: 12px; color: #9ca3af;">ID: <?php echo intval($request->product_id); ?></div>
                                    </td>
                                    <td style="padding: 20px 25px;">
                                        <div style="font-weight: 600; color: #374151; font-size: 14px;">
                                            <?php echo esc_html($request->customer_name ?: '-'); ?>
                                        </div>
                                    </td>
                                    <td style="padding: 20px 25px;">
                                        <div style="display: inline-block; padding: 6px 12px; background: #f3f4f6; border-radius: 6px; font-size: 13px; font-weight: 500; color: #4b5563;">
                                            <?php echo esc_html($request->customer_email); ?>
                                        </div>
                                    </td>
                                    <td style="padding: 20px 25px;">
                                        <div style="font-size: 13px; color: #6b7280; font-weight: 500;">
                                            <?php echo esc_html(date_i18n('d/m/Y', strtotime($request->date_requested))); ?>
                                        </div>
                                        <div style="font-size: 12px; color: #9ca3af;">
                                            <?php echo esc_html(date_i18n('H:i', strtotime($request->date_requested))); ?>
                                        </div>
                                    </td>
                                    <td style="padding: 20px 25px; text-align: center;">
                                        <?php if ($request->notified) : ?>
                                            <span style="display: inline-block; padding: 7px 16px; background: linear-gradient(135deg, #10b981, #059669); color: white; border-radius: 20px; font-size: 12px; font-weight: 700; box-shadow: 0 2px 8px rgba(16, 185, 129, 0.3);">
                                                ✅ Notifié
                                            </span>
                                        <?php else : ?>
                                            <span style="display: inline-block; padding: 7px 16px; background: linear-gradient(135deg, #f59e0b, #d97706); color: white; border-radius: 20px; font-size: 12px; font-weight: 700; box-shadow: 0 2px 8px rgba(245, 158, 11, 0.3);">
                                                ⏳ En attente
                                            </span>
                                        <?php endif; ?>
                                    </td>
                                    <td style="padding: 20px 25px; text-align: center;">
                                        <div style="display: flex; gap: 8px; justify-content: center;">
                                            <?php if ($request->product_name) : ?>
                                                <a href="<?php echo esc_url(get_edit_post_link($request->product_id)); ?>" 
                                                   style="display: inline-block; padding: 8px 14px; background: #0746C0; color: white; border-radius: 8px; font-size: 12px; font-weight: 600; text-decoration: none; transition: all 0.3s;"
                                                   onmouseover="this.style.background='#05389a'; this.style.transform='translateY(-2px)';"
                                                   onmouseout="this.style.background='#0746C0'; this.style.transform='translateY(0)';"
                                                   title="Voir le produit">
                                                    👁️ Voir
                                                </a>
                                            <?php endif; ?>
                                            <a href="<?php echo esc_url(wp_nonce_url(admin_url('admin.php?page=waitlist-manager&action=delete&id=' . $request->id), 'delete_waitlist_' . $request->id)); ?>" 
                                               onclick="return confirm('❌ Supprimer cette entrée ?');"
                                               style="display: inline-block; padding: 8px 14px; background: #ef4444; color: white; border-radius: 8px; font-size: 12px; font-weight: 600; text-decoration: none; transition: all 0.3s;"
                                               onmouseover="this.style.background='#dc2626'; this.style.transform='translateY(-2px)';"
                                               onmouseout="this.style.background='#ef4444'; this.style.transform='translateY(0)';"
                                               title="Supprimer">
                                                🗑️
                                            </a>
                                        </div>
                                    </td>
                                </tr>
                            <?php endforeach; ?>
                        <?php endif; ?>
                    </tbody>
                </table>
            </div>
            
            <!-- Pagination Moderne -->
            <?php if ($total > $per_page) : 
                $total_pages = ceil($total / $per_page);
            ?>
                <div style="padding: 25px 30px; border-top: 1px solid #f0f0f0; display: flex; justify-content: space-between; align-items: center;">
                    <div style="font-size: 14px; color: #6b7280;">
                        Affichage <strong><?php echo (($paged - 1) * $per_page + 1); ?></strong> à 
                        <strong><?php echo min($paged * $per_page, $total); ?></strong> sur 
                        <strong><?php echo intval($total); ?></strong> entrées
                    </div>
                    <div style="display: flex; gap: 8px;">
                        <?php
                        $pagination = paginate_links([
                            'base' => add_query_arg('paged', '%#%'),
                            'format' => '',
                            'current' => $paged,
                            'total' => $total_pages,
                            'prev_text' => '← Préc',
                            'next_text' => 'Suiv →',
                            'type' => 'array'
                        ]);
                        
                        if ($pagination) {
                            foreach ($pagination as $link) {
                                $link = str_replace('page-numbers', 'page-numbers-custom', $link);
                                echo wp_kses_post($link);
                            }
                        }
                        ?>
                    </div>
                </div>
                <style>
                    .page-numbers-custom {
                        display: inline-block;
                        padding: 10px 16px;
                        background: #f3f4f6;
                        color: #4b5563;
                        border-radius: 8px;
                        font-size: 14px;
                        font-weight: 600;
                        text-decoration: none;
                        transition: all 0.3s;
                    }
                    .page-numbers-custom:hover {
                        background: #e5e7eb;
                        transform: translateY(-2px);
                    }
                    .page-numbers-custom.current {
                        background: linear-gradient(135deg, #0746C0, #0532a0);
                        color: white;
                        box-shadow: 0 4px 12px rgba(7,70,192,0.3);
                    }
                </style>
            <?php endif; ?>
            
        </div>
        
        <!-- Info Box -->
        <div style="margin-top: 25px; padding: 20px 25px; background: linear-gradient(135deg, #eff6ff, #dbeafe); border-left: 4px solid #0746C0; border-radius: 12px; display: flex; align-items: start; gap: 15px;">
            <div style="font-size: 24px;">💡</div>
            <div>
                <div style="font-weight: 700; color: #0746C0; margin-bottom: 5px; font-size: 14px;">Fonctionnement automatique</div>
                <div style="color: #1e40af; font-size: 13px; line-height: 1.6;">
                    Les clients sont <strong>automatiquement notifiés par email</strong> dès que vous remettez un produit en stock. 
                    Les anciennes entrées notifiées de plus de 6 mois sont automatiquement supprimées pour optimiser les performances.
                </div>
            </div>
        </div>
    </div>
    <?php
}

// =====================================================
// 9. NETTOYAGE AUTOMATIQUE (Performance)
// =====================================================
function waitlist_schedule_cleanup() {
    if (!wp_next_scheduled('waitlist_daily_cleanup')) {
        wp_schedule_event(time(), 'daily', 'waitlist_daily_cleanup');
    }
}
add_action('wp', 'waitlist_schedule_cleanup');

function waitlist_daily_cleanup() {
    global $wpdb;
    $table_name = $wpdb->prefix . 'product_waitlist';
    
    // Supprimer les entrées notifiées de plus de 6 mois
    $wpdb->query("DELETE FROM $table_name WHERE notified = 1 AND date_requested < DATE_SUB(NOW(), INTERVAL 6 MONTH)");
    
    // Supprimer les entrées de produits supprimés
    $wpdb->query("DELETE w FROM $table_name w LEFT JOIN {$wpdb->posts} p ON w.product_id = p.ID WHERE p.ID IS NULL");
}
add_action('waitlist_daily_cleanup', 'waitlist_daily_cleanup');

// =====================================================
// 10. DÉSACTIVATION
// =====================================================
register_deactivation_hook(__FILE__, 'waitlist_deactivation');
function waitlist_deactivation() {
    wp_clear_scheduled_hook('waitlist_daily_cleanup');
}
?>
