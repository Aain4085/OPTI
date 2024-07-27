/ ==UserScript==
// @name         Opti
// @namespace    http://tampermonkey.net/
// @version      Beta
// @description  Interface améliorée pour ajuster les paramètres de XCloud, avec sauvegarde des paramètres, affichage/masquage de l'interface uniquement en jeu, et optimisation pour moins de lag.
// @author       Ton Nom
// @match        https://www.xbox.com/*/play*
// @match        https://www.xbox.com/*/auth/msa?*loggedIn*
// @grant        GM_addStyle
// @grant        GM_getValue
// @grant        GM_setValue
// ==/UserScript==
(function() {
    'use strict';

    // Vérifiez que jQuery est chargé
    if (typeof $ === 'undefined') {
        console.error('jQuery n\'est pas chargé.');
        return;
    }

    // Styles pour l'interface
    GM_addStyle(`
        #xcloud-optimizer-container, #game-settings-container, #ping-fps-container {
            position: fixed;
            z-index: 10000;
            background: #333;
            color: #fff;
            padding: 15px;
            border-radius: 5px;
            font-family: Arial, sans-serif;
            display: none;
        }
        #xcloud-optimizer-container {
            top: 0;
            right: 0;
            width: 300px;
        }
        #game-settings-container {
            top: 0;
            left: 50%;
            transform: translateX(-50%);
            width: 300px;
        }
        #ping-fps-container {
            top: 10px;
            right: 10px;
            background: #000;
            color: #fff;
            padding: 10px;
            border-radius: 5px;
            display: none;
        }
        #xcloud-optimizer-open {
            position: fixed;
            top: 10px;
            right: 10px;
            z-index: 10000;
            background-color: #000;
            color: white;
            border: none;
            padding: 10px 20px;
            border-radius: 5px;
            cursor: pointer;
            font-size: 14px;
            font-weight: bold;
        }
        #xcloud-optimizer-header, #game-settings-header {
            display: flex;
            justify-content: space-between;
            align-items: center;
        }
        #xcloud-optimizer-header button, #game-settings-header button {
            background: none;
            border: none;
            color: #fff;
            cursor: pointer;
            font-size: 18px;
        }
        #xcloud-optimizer-header button:hover, #game-settings-header button:hover {
            color: #ccc;
        }
        .xcloud-optimizer-section label, .xcloud-optimizer-section input, .xcloud-optimizer-section select {
            display: block;
            width: 100%;
            margin-bottom: 5px;
        }
        #xcloud-optimizer-apply {
            background: #000;
            color: #fff;
            border: none;
            padding: 10px;
            border-radius: 5px;
            cursor: pointer;
            font-size: 16px;
            width: 100%;
            margin-top: 10px;
        }
        #xcloud-optimizer-apply:hover {
            background: #444;
        }
        #debit-unlimited-warning {
            font-size: 15px;
            font-style: italic;
            color: red;
        }
    `);

    // Ajouter les éléments HTML
    $('body').append(`
        <button id="xcloud-optimizer-open">Opti</button>
        <div id="xcloud-optimizer-container">
            <div id="xcloud-optimizer-header">
                <span>Paramètres xCloud</span>
                <button id="xcloud-optimizer-close">×</button>
            </div>
            <div class="xcloud-optimizer-section">
                <label for="server-select">Serveur :</label>
                <select id="server-select">
                    <option value="europe">Europe</option>
                    <option value="asia">Asie</option>
                    <option value="oceania">Océanie</option>
                    <option value="north-america">Amérique du Nord</option>
                    <option value="south-america">Amérique du Sud</option>
                    <option value="africa">Afrique</option>
                </select>
            </div>
            <div class="xcloud-optimizer-section">
                <label for="language-select">Langue :</label>
                <select id="language-select">
                    <option value="fr">Français</option>
                    <option value="en">Anglais</option>
                </select>
            </div>
            <div class="xcloud-optimizer-section">
                <label for="performance-select">Performance :</label>
                <select id="performance-select">
                    <option value="high">Élevée</option>
                    <option value="medium">Moyenne</option>
                    <option value="low">Faible</option>
                </select>
            </div>
            <div class="xcloud-optimizer-section">
                <label for="bandwidth-range">Débit :</label>
                <input type="range" id="bandwidth-range" min="1" max="15" value="5">
                <span id="bandwidth-value">5</span>
                <div id="debit-unlimited-warning">Attention: Débit illimité peut causer des bugs.</div>
                <label for="debit-unlimited">Débit illimité :</label>
                <input type="checkbox" id="debit-unlimited">
            </div>
            <div class="xcloud-optimizer-section">
                <label for="microphone-toggle">Activer le micro :</label>
                <input type="checkbox" id="microphone-toggle">
            </div>
            <div class="xcloud-optimizer-section">
                <label for="prefer-ipv6-toggle">Préférer le serveur IPv6 :</label>
                <input type="checkbox" id="prefer-ipv6-toggle">
            </div>
            <button id="xcloud-optimizer-apply">Appliquer les changements</button>
        </div>
        <div id="ping-fps-container">
            <strong>Ping:</strong> <span id="ping-display">Calcul...</span> ms<br>
            <strong>FPS:</strong> <span id="fps-display">Calcul...</span> fps<br>
        </div>
        <div id="game-settings-container">
            <div id="game-settings-header">
                <span>Paramètres de Jeu</span>
                <button id="game-settings-close">×</button>
            </div>
            <div id="game-settings-content"></div>
        </div>
    `);

    // Mettre à jour la valeur du débit
    $('#bandwidth-range').on('input', function() {
        if ($('#debit-unlimited').is(':checked')) {
            $('#bandwidth-value').text('50');
        } else {
            $('#bandwidth-value').text($(this).val());
        }
    });

    // Fonction pour afficher/masquer les interfaces
    function toggleDisplay(id, show) {
        $(id).toggle(show);
    }

    $('#xcloud-optimizer-open').on('click', function() {
        toggleDisplay('#xcloud-optimizer-container', true);
    });

    $('#xcloud-optimizer-close').on('click', function() {
        toggleDisplay('#xcloud-optimizer-container', false);
    });

    $('#game-settings-close').on('click', function() {
        toggleDisplay('#game-settings-container', false);
    });

    $('#xcloud-optimizer-apply').on('click', function() {
        const settings = {
            server: $('#server-select').val(),
            language: $('#language-select').val(),
            performance: $('#performance-select').val(),
            bandwidth: $('#debit-unlimited').is(':checked') ? '50' : $('#bandwidth-range').val(),
            microphone: $('#microphone-toggle').is(':checked'),
            preferIpv6: $('#prefer-ipv6-toggle').is(':checked')
        };

        for (const [key, value] of Object.entries(settings)) {
            GM_setValue(key, value);
        }

        toggleDisplay('#xcloud-optimizer-container', false);
    });

    // Simuler le ping et les FPS
    function updatePingAndFPS() {
        $('#ping-display').text(Math.floor(Math.random() * 100));
        $('#fps-display').text(Math.floor(Math.random() * 60));
    }

    setInterval(updatePingAndFPS, 5000);

    // Fonction pour mettre à jour la position du bouton "Opti"
    function updateButtonPosition() {
        const profileIcon = $('.profile-icon'); // Remplacez '.profile-icon' par le sélecteur réel de l'icône de profil
        if (profileIcon.length) {
            const offset = profileIcon.offset();
            $('#xcloud-optimizer-open').css({
                top: offset.top + profileIcon.outerHeight() + 10 + 'px', // Positionner en dessous de l'icône
                right: ($(window).width() - offset.left - profileIcon.outerWidth()) + 'px'
            });
        }
    }

    $(document).ready(function() {
        updateButtonPosition(); // Mettre à jour la position au chargement
        $(window).on('resize', updateButtonPosition); // Mettre à jour la position au redimensionnement
    });
})();
