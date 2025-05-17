# First-plugin-parking-lot-system-
<?php
/*
Plugin Name: Smart Parking System
Description: Real-time smart parking with reserve and park-now functionality.
Version: 1.1
Author: Your Name
*/

if (!defined('ABSPATH')) exit;

// Enqueue JS/CSS Inline
add_action('wp_footer', function () {
    if (!is_singular()) return;
    ?>
    <style>
        .slot {
            border: 2px solid #ccc;
            padding: 10px;
            margin: 10px;
            width: 220px;
            text-align: center;
            border-radius: 12px;
            background: #f9f9f9;
            display: inline-block;
        }
        .status-Free { background-color: #d4edda; }
        .status-Reserved { background-color: #fff3cd; }
        .status-Parked { background-color: #f8d7da; }
        .slot button {
            margin: 5px;
            padding: 8px 12px;
            border: none;
            border-radius: 6px;
            cursor: pointer;
        }
        .reserve-btn { background: #ffc107; color: black; }
        .park-btn { background: #28a745; color: white; }
        .parknow-btn { background: #007bff; color: white; }
    </style>
    <script>
        document.addEventListener('DOMContentLoaded', function () {
            const currentUser = <?php echo get_current_user_id() ?: 0; ?>;

            function loadSlots() {
                fetch('<?php echo admin_url('admin-ajax.php'); ?>', {
                    method: 'POST',
                    headers: {'Content-Type': 'application/x-www-form-urlencoded'},
                    body: 'action=sps_get_slots'
                })
                .then(res => res.json())
                .then(res => {
                    if (!res.success) return;
                    const slots = res.data;
                    document.querySelectorAll('.slot').forEach(slot => {
                        const slotId = slot.dataset.slotId;
                        const data = slots[slotId];
                        slot.className = 'slot status-' + data.status;
                        slot.querySelector('.status-text').innerHTML = `<strong>${data.status}</strong>`;
                        const timerDiv = slot.querySelector('.timer');
                        const actionsDiv = slot.querySelector('.actions');
                        timerDiv.innerHTML = '';
                        actionsDiv.innerHTML = '';
                        const left = data.expires_at - Math.floor(Date.now() / 1000);
                        if (left > 0) {
                            const mins = Math.floor(left / 60);
                            const secs = left % 60;
                            timerDiv.textContent = `⏳ ${mins}m ${secs}s`;
                        }
                        if (data.status === 'Free') {
                            if(currentUser !== 0){
                                actionsDiv.innerHTML = `
                                    <button class="reserve-btn" onclick="reserveSlot(${slotId})">Reserve</button>
                                    <button class="park-btn" onclick="promptPark(${slotId})">Park</button>`;
                            } else {
                                actionsDiv.innerHTML = `<div>Please login to Reserve or Park</div>`;
                            }
                        } else if (data.status === 'Reserved') {
                            if (data.user_id == currentUser && currentUser !== 0) {
                                actionsDiv.innerHTML = `
                                    <button class="parknow-btn" onclick="promptParkNow(${slotId})">Park Now</button>
                                    <div>Reserved by you</div>`;
                            } else {
                                actionsDiv.innerHTML = `<div>Reserved by someone else</div>`;
                            }
                        } else if (data.status === 'Parked') {
                            actionsDiv.innerHTML = `<div>🚗 Parked</div>`;
                        }
                    });
                });
            }

            window.reserveSlot = function (slotId) {
                fetch('<?php echo admin_url('admin-ajax.php'); ?>', {
                    method: 'POST',
                    headers: {'Content-Type': 'application/x-www-form-urlencoded'},
                    body: `action=sps_reserve&slot_id=${slotId}`
                }).then(() => loadSlots());
            };

            window.promptPark = function (slotId) {
                let mins = prompt('Enter duration in minutes:');
                if (!mins || isNaN(mins)) return;
                fetch('<?php echo admin_url('admin-ajax.php'); ?>', {
                    method: 'POST',
                    headers: {'Content-Type': 'application/x-www-form-urlencoded'},
                    body: `action=sps_park&slot_id=${slotId}&duration=${mins}`
                }).then(() => loadSlots());
            };

            window.promptParkNow = function (slotId) {
                let mins = prompt('Enter duration in minutes:');
                if (!mins || isNaN(mins)) return;
                fetch('<?php echo admin_url('admin-ajax.php'); ?>', {
                    method: 'POST',
                    headers: {'Content-Type': 'application/x-www-form-urlencoded'},
                    body: `action=sps_parknow&slot_id=${slotId}&duration=${mins}`
                }).then(() => loadSlots());
            };

            loadSlots();
            setInterval(loadSlots, 1000);
        });
    </script>
    <?php
});

// Shortcode to render UI (with 10 slots)
add_shortcode('smart_parking_system', function () {
    ob_start();
    $slots = get_option('sps_parking_slots');
    if (!$slots) {
        $slots = [];
        for ($i = 1; $i <= 10; $i++) {
            $slots[$i] = ['status' => 'Free', 'user_id' => 0, 'expires_at' => 0];
        }
        update_option('sps_parking_slots', $slots);
    }

    echo '<div id="parking-lot">';
    foreach ($slots as $id => $slot) {
        echo "<div class='slot status-{$slot['status']}' data-slot-id='{$id}'>
                <h3>Slot {$id}</h3>
                <div class='status-text'><strong>{$slot['status']}</strong></div>
                <div class='timer'></div>
                <div class='actions'></div>
              </div>";
    }
    echo '</div>';
    return ob_get_clean();
});

// AJAX handlers
add_action('wp_ajax_sps_get_slots', 'sps_get_slots');
add_action('wp_ajax_nopriv_sps_get_slots', 'sps_get_slots');
function sps_get_slots() {
    $slots = get_option('sps_parking_slots');
    $now = time();
    foreach ($slots as $id => $slot) {
        if ($slot['expires_at'] > 0 && $slot['expires_at'] <= $now) {
            $slots[$id] = ['status' => 'Free', 'user_id' => 0, 'expires_at' => 0];
        }
    }
    update_option('sps_parking_slots', $slots);
    wp_send_json_success($slots);
}

add_action('wp_ajax_sps_reserve', 'sps_reserve');
function sps_reserve() {
    if (!is_user_logged_in()) wp_send_json_error('Login required');
    $id = intval($_POST['slot_id']);
    $slots = get_option('sps_parking_slots');
    if (isset($slots[$id]) && $slots[$id]['status'] === 'Free') {
        $slots[$id] = [
            'status' => 'Reserved',
            'user_id' => get_current_user_id(),
            'expires_at' => time() + 300 // 5 min reservation
        ];
        update_option('sps_parking_slots', $slots);
        wp_send_json_success();
    }
    wp_send_json_error('Slot not free');
}

add_action('wp_ajax_sps_park', 'sps_park');
function sps_park() {
    if (!is_user_logged_in()) wp_send_json_error('Login required');
    $id = intval($_POST['slot_id']);
    $mins = intval($_POST['duration']);
    $slots = get_option('sps_parking_slots');
    if (isset($slots[$id]) && $slots[$id]['status'] === 'Free') {
        $slots[$id] = [
            'status' => 'Parked',
            'user_id' => get_current_user_id(),
            'expires_at' => time() + ($mins * 60)
        ];
        update_option('sps_parking_slots', $slots);
        wp_send_json_success();
    }
    wp_send_json_error('Slot not free');
}

add_action('wp_ajax_sps_parknow', 'sps_parknow');
function sps_parknow() {
    if (!is_user_logged_in()) wp_send_json_error('Login required');
    $id = intval($_POST['slot_id']);
    $mins = intval($_POST['duration']);
    $uid = get_current_user_id();
    $slots = get_option('sps_parking_slots');
    if (isset($slots[$id]) && $slots[$id]['status'] === 'Reserved' && $slots[$id]['user_id'] == $uid) {
        $slots[$id]['status'] = 'Parked';
        $slots[$id]['expires_at'] = time() + ($mins * 60);
        update_option('sps_parking_slots', $slots);
        wp_send_json_success();
    }
    wp_send_json_error('Cannot park now');
}

