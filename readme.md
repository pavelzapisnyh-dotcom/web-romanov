import sys
import traceback

try:
    import sqlite3
    import hashlib
    import io
    import csv
    import re
    import webbrowser
    import threading
    import os
    from datetime import datetime, timedelta
    from functools import wraps

    if getattr(sys, 'frozen', False):
        sys.path.insert(0, os.path.dirname(sys.executable))

    from flask import Flask, render_template_string, request, redirect, url_for, session, flash, jsonify, send_file

    if getattr(sys, 'frozen', False):
        base_dir = os.path.dirname(sys.executable)
    else:
        base_dir = os.path.dirname(os.path.abspath(__file__))
    os.chdir(base_dir)

    def excepthook(exc_type, exc_value, exc_traceback):
        traceback.print_exception(exc_type, exc_value, exc_traceback)
        input("\nНажмите Enter для выхода...")
        sys.exit(1)
    sys.excepthook = excepthook

    app = Flask(__name__)
    app.secret_key = 'souzidatel_secret_key_2025'

    BASE_HTML = '''
    <!DOCTYPE html>
    <html lang="ru">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>ООО «Созидатель»</title>
        <style>
            * { box-sizing: border-box; }
            body { background: #f0fdf4; margin: 0; font-family: system-ui, -apple-system, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif; }
            .container { max-width: 1200px; margin: 0 auto; padding: 0 1rem; }
            .header { background: #166534; color: white; box-shadow: 0 4px 6px -1px rgba(0,0,0,0.1); position: relative; overflow: hidden; }
            .header-bg { position: absolute; inset: 0; background: repeating-linear-gradient(45deg, rgba(255,255,255,0.05) 0px, rgba(255,255,255,0.05) 20px, transparent 20px, transparent 40px); pointer-events: none; }
            .header-inner { display: flex; justify-content: space-between; align-items: center; flex-wrap: wrap; padding: 0.75rem 0; position: relative; }
            .logo { font-size: 1.25rem; font-weight: bold; margin: 0; }
            .user-area { display: flex; gap: 1rem; align-items: center; }
            .user-area a { color: white; text-decoration: none; }
            .user-area a:hover { text-decoration: underline; }
            .admin-tabs { margin-top: 0.5rem; display: flex; gap: 1.5rem; border-top: 1px solid #15803d; padding-top: 0.5rem; }
            .tab-btn { background: none; border: none; color: white; cursor: pointer; font-size: 1rem; padding: 0.25rem 0; transition: color 0.2s; }
            .tab-btn:hover { color: #fde047; }
            .tab-active { border-bottom: 2px solid #fbbf24; color: #fbbf24; }
            .card { background: white; border-radius: 0.5rem; box-shadow: 0 1px 3px 0 rgba(0,0,0,0.1); padding: 1.5rem; margin-bottom: 1.5rem; }
            .grid { display: grid; gap: 1rem; }
            .grid-cols-2 { grid-template-columns: repeat(2, 1fr); }
            .grid-cols-5 { grid-template-columns: repeat(5, 1fr); }
            @media (min-width: 768px) { .md-grid-cols-3 { grid-template-columns: repeat(3, 1fr); } .md-grid-cols-5 { grid-template-columns: repeat(5, 1fr); } }
            table { width: 100%; border-collapse: collapse; }
            th, td { border: 1px solid #e5e7eb; padding: 0.5rem; text-align: left; }
            th { background-color: #f3f4f6; }
            .btn { display: inline-block; padding: 0.25rem 0.75rem; border-radius: 0.25rem; font-size: 0.875rem; cursor: pointer; border: none; }
            .btn-sm { padding: 0.125rem 0.5rem; font-size: 0.75rem; }
            .btn-blue { background: #3b82f6; color: white; }
            .btn-green { background: #16a34a; color: white; }
            .btn-red { background: #ef4444; color: white; }
            .btn-gray { background: #6b7280; color: white; }
            .btn-yellow { background: #eab308; color: white; }
            .btn-purple { background: #9333ea; color: white; }
            .btn:hover { filter: brightness(0.9); }
            input, select, textarea { width: 100%; padding: 0.5rem; border: 1px solid #d1d5db; border-radius: 0.25rem; margin-bottom: 0.5rem; }
            .flash { background: #fef9c3; border-left: 4px solid #eab308; padding: 0.5rem; margin-bottom: 0.5rem; }
            .mt-2 { margin-top: 0.5rem; }
            .mb-2 { margin-bottom: 0.5rem; }
            .mb-4 { margin-bottom: 1rem; }
            .p-2 { padding: 0.5rem; }
            .p-4 { padding: 1rem; }
            .text-center { text-align: center; }
            .text-sm { font-size: 0.875rem; }
            .font-bold { font-weight: bold; }
            .flex { display: flex; }
            .flex-wrap { flex-wrap: wrap; }
            .gap-2 { gap: 0.5rem; }
            .gap-3 { gap: 0.75rem; }
            .justify-between { justify-content: space-between; }
            .items-center { align-items: center; }
            .border { border: 1px solid #e5e7eb; }
            .rounded { border-radius: 0.25rem; }
            .bg-white { background: white; }
            .bg-green-100 { background: #dcfce7; }
            .bg-yellow-100 { background: #fef9c3; }
            .bg-blue-100 { background: #dbeafe; }
            .bg-purple-100 { background: #f3e8ff; }
            .bg-red-100 { background: #fee2e2; }
            .whitespace-nowrap { white-space: nowrap; }
            .overflow-x-auto { overflow-x: auto; }
            .hidden { display: none; }
            .inline-block { display: inline-block; }

            /* Стили для фото */
            .company-photo {
                width: 100%;
                max-height: 400px;
                object-fit: cover;
                border-radius: 0.5rem;
                box-shadow: 0 4px 12px rgba(0,0,0,0.1);
            }
            .photo-container {
                position: relative;
                margin-bottom: 1.5rem;
            }
            .photo-caption {
                position: absolute;
                bottom: 1rem;
                right: 1rem;
                background: rgba(22, 101, 52, 0.85);
                color: white;
                padding: 0.5rem 1rem;
                border-radius: 0.5rem;
                font-size: 0.875rem;
            }

            /* Стили для карточек услуг */
            .service-card {
                transition: transform 0.2s, box-shadow 0.2s;
                cursor: default;
            }
            .service-card:hover {
                transform: translateY(-4px);
                box-shadow: 0 10px 25px rgba(0,0,0,0.1);
            }
            .service-icon {
                font-size: 3rem;
                display: block;
                margin-bottom: 0.5rem;
            }
        </style>
    </head>
    <body>
        <div class="header">
            <div class="header-bg"></div>
            <div class="container">
                <div class="header-inner">
                    <h1 class="logo">🌱 ООО «Созидатель»</h1>
                    <div class="user-area">
                        {% if session.user_id %}
                            <span>👤 {{ session.full_name }} ({{ session.role }})</span>
                            <a href="/logout">🚪 Выйти</a>
                        {% else %}
                            <a href="/login">🔐 Вход</a>
                            <a href="/register">📝 Регистрация</a>
                        {% endif %}
                    </div>
                </div>
                {% if session.user_id and session.role == 'admin' %}
                <div class="admin-tabs">
                    <button class="tab-btn" data-tab="requests">📋 Заявки</button>
                    <button class="tab-btn" data-tab="resources">📦 Ресурсы</button>
                    <button class="tab-btn" data-tab="seasons">🌿 Сезоны</button>
                </div>
                {% endif %}
            </div>
        </div>
        <main class="container" style="padding: 1rem;">
            {% with messages = get_flashed_messages() %}{% if messages %}<div class="mb-4">{% for msg in messages %}<div class="flash">{{ msg }}</div>{% endfor %}</div>{% endif %}{% endwith %}
            {{ content | safe }}
        </main>
        <script>
            const tabBtns = document.querySelectorAll('.tab-btn');
            const tabContents = document.querySelectorAll('.tab-content');
            if (tabBtns.length) {
                function switchTab(tabId) {
                    tabContents.forEach(c => c.classList.add('hidden'));
                    const target = document.getElementById(`tab-${tabId}`);
                    if (target) target.classList.remove('hidden');
                    tabBtns.forEach(btn => btn.classList.remove('tab-active'));
                    const activeBtn = Array.from(tabBtns).find(btn => btn.dataset.tab === tabId);
                    if (activeBtn) activeBtn.classList.add('tab-active');
                }
                tabBtns.forEach(btn => btn.addEventListener('click', () => switchTab(btn.dataset.tab)));
                if (tabContents.length) switchTab('requests');
            }
        </script>
    </body>
    </html>
    '''

    # ------------------- СТРАНИЦЫ (HTML) -------------------
    WELCOME_PAGE = '''
    <div style="position:relative; padding:0 2rem;">
        <!-- Левая декоративная полоса -->
        <div style="position:absolute; left:0; top:0; bottom:0; width:6px; background: linear-gradient(to bottom, #166534, #22c55e, #86efac, #22c55e, #166534); border-radius:0 6px 6px 0; box-shadow: 2px 0 10px rgba(22,101,52,0.1);"></div>

        <!-- Правая декоративная полоса -->
        <div style="position:absolute; right:0; top:0; bottom:0; width:6px; background: linear-gradient(to bottom, #166534, #22c55e, #86efac, #22c55e, #166534); border-radius:6px 0 0 6px; box-shadow: -2px 0 10px rgba(22,101,52,0.1);"></div>

        <!-- Декоративные листья слева (верх) -->
        <div style="position:absolute; left:12px; top:20px; opacity:0.3;">
            <svg width="30" height="30" viewBox="0 0 30 30" fill="none">
                <path d="M15 5 Q20 10 25 15 Q20 20 15 25 Q10 20 5 15 Q10 10 15 5Z" fill="#166534"/>
            </svg>
        </div>

        <!-- Декоративные листья слева (низ) -->
        <div style="position:absolute; left:12px; bottom:20px; opacity:0.3;">
            <svg width="30" height="30" viewBox="0 0 30 30" fill="none">
                <path d="M15 5 Q20 10 25 15 Q20 20 15 25 Q10 20 5 15 Q10 10 15 5Z" fill="#166534"/>
            </svg>
        </div>

        <!-- Декоративные листья справа (верх) -->
        <div style="position:absolute; right:12px; top:20px; opacity:0.3;">
            <svg width="30" height="30" viewBox="0 0 30 30" fill="none">
                <path d="M15 5 Q20 10 25 15 Q20 20 15 25 Q10 20 5 15 Q10 10 15 5Z" fill="#166534"/>
            </svg>
        </div>

        <!-- Декоративные листья справа (низ) -->
        <div style="position:absolute; right:12px; bottom:20px; opacity:0.3;">
            <svg width="30" height="30" viewBox="0 0 30 30" fill="none">
                <path d="M15 5 Q20 10 25 15 Q20 20 15 25 Q10 20 5 15 Q10 10 15 5Z" fill="#166534"/>
            </svg>
        </div>

        <div class="card" style="position:relative; z-index:1; padding:1.5rem 2rem;">
            <!-- Заголовок с логотипом -->
            <div style="display:flex; justify-content:space-between; align-items:center; margin-bottom:1rem; flex-wrap:wrap;">
                <div>
                    <h1 style="font-size:2rem; color:#166534; margin:0;">ООО «Созидатель»</h1>
                    <h2 style="font-size:1.2rem; color:#1f2937; margin:0.25rem 0 0 0;">САДОВЫЙ ЦЕНТР В ВОЛГОДОНСКЕ</h2>
                    <p style="font-size:0.95rem; color:#6b7280; margin:0.25rem 0 0 0;">Ваш надёжный партнёр в сфере растениеводства, озеленения и ландшафтного дизайна</p>
                </div>
                <!-- Логотип -->
                <div style="flex-shrink:0; margin-left:1rem;">
                    <svg width="80" height="80" viewBox="0 0 100 100" fill="none" xmlns="http://www.w3.org/2000/svg">
                        <circle cx="50" cy="50" r="45" fill="#dcfce7" stroke="#166534" stroke-width="3"/>
                        <path d="M30 50 Q35 30 50 35 Q40 40 35 50 Q30 55 30 50Z" fill="#22c55e"/>
                        <path d="M70 50 Q65 30 50 35 Q60 40 65 50 Q70 55 70 50Z" fill="#22c55e"/>
                        <path d="M50 20 L50 60 L45 45 L40 50 L45 40 L35 35 L45 30 L40 20 L50 25 L60 20 L55 30 L65 35 L55 40 L60 50 L55 45 L50 60Z" fill="#15803d"/>
                        <path d="M50 60 L50 80" stroke="#166534" stroke-width="3" stroke-linecap="round"/>
                        <path d="M50 70 Q45 65 47 60" stroke="#166534" stroke-width="2" fill="none" stroke-linecap="round"/>
                        <path d="M50 70 Q55 65 53 60" stroke="#166534" stroke-width="2" fill="none" stroke-linecap="round"/>
                        <text x="50" y="52" font-family="Arial, sans-serif" font-size="18" font-weight="bold" fill="white" text-anchor="middle">С</text>
                    </svg>
                </div>
            </div>

            <!-- Фото компании -->
            <div style="margin: 1.5rem 0;">
                <img src="/static/organization.png" alt="ООО Созидатель" style="width:100%; max-height:400px; object-fit:cover; border-radius:0.5rem; box-shadow:0 4px 12px rgba(0,0,0,0.1);">
            </div>

            <!-- Надпись "Подробнее о компании" -->
            <div style="text-align:left; margin-bottom:1rem; padding:0.5rem 0; border-bottom:2px solid #dcfce7;">
                <span style="font-size:1.1rem; color:#166534; font-weight:bold; cursor:pointer;">
                    📖 Подробнее о компании
                </span>
            </div>

            <!-- Описание -->
            <div style="background:#dcfce7; border-radius:0.5rem; padding:1.5rem; margin-bottom:1.5rem;">
                <p style="font-size:1.05rem; color:#1f2937; margin:0 0 0.5rem 0;">
                    🌿 Продажа растений от саженцев до взрослых крупномерных хвойных и лиственных деревьев.
                </p>
                <p style="font-size:1.05rem; color:#1f2937; margin:0 0 0.5rem 0;">
                    🌱 Наши растения выращены в собственном питомнике и адаптированы под климат Ростовской области!
                </p>
                <p style="font-size:1.05rem; color:#1f2937; margin:0 0 0.5rem 0;">
                    🏡 Рулонный газон, садовый инструмент, садовая мебель.
                </p>
                <p style="font-size:1.05rem; color:#1f2937; margin:0;">
                    🪨 Природный камень, малые архитектурные формы, декоративные элементы.
                </p>
            </div>

            <!-- Раздел "Наши услуги" -->
            <h3 style="color:#166534; font-size:1.3rem; margin:1.5rem 0 0.5rem 0;">Нашим клиентам доступен широкий выбор товаров и услуг:</h3>
            <ul style="list-style:none; padding:0; display:grid; grid-template-columns:repeat(auto-fit, minmax(280px, 1fr)); gap:0.5rem; margin-bottom:1.5rem;">
                <li style="background:#f8fafc; padding:0.5rem 1rem; border-radius:0.25rem; border-left:3px solid #16a34a;">🌱 Огромный выбор растений от саженцев до взрослых крупномерных деревьев</li>
                <li style="background:#f8fafc; padding:0.5rem 1rem; border-radius:0.25rem; border-left:3px solid #16a34a;">🧪 Широкий выбор удобрений и средств по уходу за растениями</li>
                <li style="background:#f8fafc; padding:0.5rem 1rem; border-radius:0.25rem; border-left:3px solid #16a34a;">💧 Водные растения для прудов и водоёмов</li>
                <li style="background:#f8fafc; padding:0.5rem 1rem; border-radius:0.25rem; border-left:3px solid #16a34a;">🔧 Садовый инструмент от секаторов до газонокосилок</li>
                <li style="background:#f8fafc; padding:0.5rem 1rem; border-radius:0.25rem; border-left:3px solid #16a34a;">🪑 Садовая мебель, беседки и декоративные элементы</li>
                <li style="background:#f8fafc; padding:0.5rem 1rem; border-radius:0.25rem; border-left:3px solid #16a34a;">🌿 Рулонный газон, семена газонной травы и средства по уходу</li>
                <li style="background:#f8fafc; padding:0.5rem 1rem; border-radius:0.25rem; border-left:3px solid #16a34a;">⚽ Уличные спортивные и детские площадки</li>
                <li style="background:#f8fafc; padding:0.5rem 1rem; border-radius:0.25rem; border-left:3px solid #16a34a;">💦 Комплектующие для систем автоматического полива</li>
                <li style="background:#f8fafc; padding:0.5rem 1rem; border-radius:0.25rem; border-left:3px solid #16a34a;">🪨 Природный камень разнообразных расцветок</li>
                <li style="background:#f8fafc; padding:0.5rem 1rem; border-radius:0.25rem; border-left:3px solid #16a34a;">🏛️ Малые архитектурные формы (МАФ)</li>
                <li style="background:#f8fafc; padding:0.5rem 1rem; border-radius:0.25rem; border-left:3px solid #16a34a;">🎨 Ландшафтный дизайн и озеленение участков</li>
                <li style="background:#f8fafc; padding:0.5rem 1rem; border-radius:0.25rem; border-left:3px solid #16a34a;">✨ И многое другое...</li>
            </ul>

            <!-- Информация о консультантах -->
            <div style="background:#fef9c3; border-radius:0.5rem; padding:1rem; margin-bottom:1.5rem; border-left:4px solid #eab308;">
                <p style="margin:0; color:#1f2937;">
                    👨‍🌾 Квалифицированные продавцы-консультанты помогут с выбором и ответят на Ваши вопросы.
                </p>
                <p style="margin:0.5rem 0 0 0; color:#1f2937;">
                    📦 Можно заказать товар с доставкой в нужное место, а растения — с посадкой и гарантией приживаемости.
                </p>
            </div>

            <!-- Наши преимущества -->
            <div style="background:#f0fdf4; border-radius:0.5rem; padding:1rem; margin-bottom:1.5rem; border:1px solid #bbf7d0;">
                <h3 style="color:#166534; margin:0 0 0.5rem 0;">⭐ Наши преимущества</h3>
                <div style="display:grid; grid-template-columns:repeat(auto-fit, minmax(200px,1fr)); gap:0.5rem;">
                    <div style="padding:0.25rem 0;">✅ Более 10 лет на рынке услуг озеленения</div>
                    <div style="padding:0.25rem 0;">✅ Собственный питомник растений</div>
                    <div style="padding:0.25rem 0;">✅ Современная техника и опытные сотрудники</div>
                    <div style="padding:0.25rem 0;">✅ Гарантия на все виды работ</div>
                    <div style="padding:0.25rem 0;">✅ Индивидуальный подход к каждому клиенту</div>
                    <div style="padding:0.25rem 0;">✅ Работаем в Волгодонске и области</div>
                </div>
            </div>

            <!-- Контактная информация -->
            <div style="background:#166534; color:white; border-radius:0.5rem; padding:1.5rem; margin-bottom:1.5rem;">
                <h3 style="color:#fbbf24; margin:0 0 0.5rem 0;">📞 Контактная информация</h3>
                <p style="margin:0.25rem 0;"><strong>📍 Юридический адрес:</strong> 347368, Ростовская область, г. Волгодонск, ул. Радужная, д. 11</p>
                <p style="margin:0.25rem 0;"><strong>📞 Телефон:</strong> +7 (918) 509-31-49</p>
                <p style="margin:0.25rem 0;"><strong>📧 E-mail:</strong> info@souzidatel.ru</p>
                <p style="margin:0.25rem 0;"><strong>🕐 Часы работы:</strong> Пн-Пт 9:00–18:00, Сб 9:00–15:00</p>
                <p style="margin:0.25rem 0;">Все вопросы по тел.: <strong>+7 (928) 279-15-10</strong></p>
            </div>

            <!-- Реквизиты компании -->
            <div style="background:#f8fafc; border-radius:0.5rem; padding:1.5rem; margin-bottom:1.5rem; border:1px solid #e5e7eb;">
                <h3 style="color:#166534; margin:0 0 0.5rem 0;">📋 Реквизиты ООО «Созидатель»</h3>
                <div style="display:grid; grid-template-columns:repeat(auto-fit, minmax(250px,1fr)); gap:0.5rem;">
                    <div><strong>ИНН:</strong> 6143053989</div>
                    <div><strong>ОГРН:</strong> 1036143005018</div>
                    <div><strong>Основной вид деятельности:</strong> Предоставление услуг в области растениеводства</div>
                </div>
                <p style="margin:0.5rem 0 0 0; font-size:0.9rem; color:#6b7280;">
                    Для получения дополнительной информации, вы можете обратиться к официальным источникам или юридическим консультантам.
                </p>
            </div>

            <!-- Нижняя информация -->
            <div style="display:grid; grid-template-columns:repeat(auto-fit, minmax(200px,1fr)); gap:1rem; background:#f8fafc; border-radius:0.5rem; padding:1rem;">
                <div>
                    <h4 style="color:#166534; margin:0 0 0.25rem 0;">🏢 ООО «Созидатель»</h4>
                    <p style="margin:0.1rem 0; font-size:0.9rem;">ул. Радужная, 11, Волгодонск</p>
                    <p style="margin:0.1rem 0; font-size:0.9rem;">Ростовская обл., Россия, 347368</p>
                    <p style="margin:0.1rem 0; font-size:0.9rem;">📞 +7 (918) 509-31-49</p>
                </div>
                <div>
                    <h4 style="color:#166534; margin:0 0 0.25rem 0;">🕐 Режим работы</h4>
                    <p style="margin:0.1rem 0; font-size:0.9rem;">Пн-Пт: 9:00 - 18:00</p>
                    <p style="margin:0.1rem 0; font-size:0.9rem;">Сб: 9:00 - 15:00</p>
                    <p style="margin:0.1rem 0; font-size:0.9rem;">Вс: Выходной</p>
                </div>
                <div>
                    <h4 style="color:#166534; margin:0 0 0.25rem 0;">📧 Связь с нами</h4>
                    <p style="margin:0.1rem 0; font-size:0.9rem;">📞 +7 (918) 509-31-49</p>
                    <p style="margin:0.1rem 0; font-size:0.9rem;">📧 info@souzidatel.ru</p>
                    <p style="margin:0.1rem 0; font-size:0.9rem;">ИНН: 6143053989</p>
                </div>
            </div>
        </div>
    </div>
    '''

    LOGIN_PAGE = '''
    <div style="max-width: 24rem; margin: 2rem auto;">
        <div class="card text-center">
            <svg width="70" height="70" viewBox="0 0 100 100" fill="none" xmlns="http://www.w3.org/2000/svg" style="margin:0 auto 0.5rem;">
                <circle cx="50" cy="50" r="45" fill="#dcfce7" stroke="#22c55e" stroke-width="2"/>
                <path d="M50 20 L55 40 L50 35 L45 40 L50 20Z" fill="#22c55e"/>
                <path d="M50 80 L55 60 L50 65 L45 60 L50 80Z" fill="#22c55e"/>
                <path d="M25 45 L45 50 L40 55 L25 45Z" fill="#22c55e"/>
                <path d="M75 55 L55 50 L60 45 L75 55Z" fill="#22c55e"/>
                <circle cx="50" cy="50" r="8" fill="#166534"/>
                <path d="M50 42 L53 50 L50 58 L47 50 L50 42Z" fill="#fbbf24"/>
            </svg>
            <h2 class="font-bold" style="font-size:1.5rem;">Вход в систему</h2>
            <form method="post" action="/login">
                <input type="text" name="username" placeholder="Логин" required>
                <input type="password" name="password" placeholder="Пароль" required>
                <button type="submit" class="btn btn-green" style="width:100%;">Войти</button>
            </form>
            <p style="margin-top: 1rem;">Нет аккаунта? <a href="/register">Зарегистрироваться</a></p>
        </div>
    </div>
    '''

    REGISTER_PAGE = '''
    <div style="max-width: 24rem; margin: 2rem auto;">
        <div class="card">
            <h2 class="font-bold" style="font-size:1.5rem; text-align:center;">Регистрация</h2>
            <form method="post" action="/register">
                <input type="text" name="full_name" placeholder="Ваше полное имя" required>
                <input type="text" name="username" placeholder="Логин" required>
                <input type="password" name="password" placeholder="Пароль" required>
                <select name="role">
                    <option value="client">Клиент</option>
                    <option value="admin">Администратор</option>
                    <option value="worker">Исполнитель</option>
                </select>
                <button type="submit" class="btn btn-green" style="width:100%;">Зарегистрироваться</button>
            </form>
            <p class="text-center" style="margin-top: 0.75rem;">Уже есть аккаунт? <a href="/login">Войти</a></p>
        </div>
    </div>
    '''

    ADMIN_PANEL_HTML = '''
    <div style="position:relative; padding:0 0.5rem;">
        <div style="height:4px; background: linear-gradient(to right, #166534, #22c55e, #86efac, #22c55e, #166534); border-radius:4px; margin-bottom:1.5rem;"></div>
        <div id="tab-requests" class="tab-content">
            <h2 style="font-size:1.5rem; color:#166534; margin:0;">📋 Управление заявками</h2>
            <p style="color:#6b7280; margin:0.25rem 0 0 0; font-size:0.9rem;">Просмотр и управление всеми заявками клиентов</p>
            <div class="grid grid-cols-5" style="gap:0.75rem; margin-top:1.5rem; margin-bottom:1.5rem;">
                <div style="background:linear-gradient(135deg, #eff6ff, #dbeafe); border-radius:0.75rem; padding:0.75rem; text-align:center; box-shadow:0 2px 8px rgba(59,130,246,0.1);">
                    <div style="font-size:1.8rem; font-weight:bold; color:#1e40af;"><span id="statTotal">0</span></div>
                    <div style="font-size:0.8rem; color:#475569;">📋 Всего</div>
                </div>
                <div style="background:linear-gradient(135deg, #fef9c3, #fef08a); border-radius:0.75rem; padding:0.75rem; text-align:center; box-shadow:0 2px 8px rgba(234,179,8,0.1);">
                    <div style="font-size:1.8rem; font-weight:bold; color:#854d0e;"><span id="statNew">0</span></div>
                    <div style="font-size:0.8rem; color:#475569;">🆕 Новых</div>
                </div>
                <div style="background:linear-gradient(135deg, #f3e8ff, #e9d5ff); border-radius:0.75rem; padding:0.75rem; text-align:center; box-shadow:0 2px 8px rgba(147,51,234,0.1);">
                    <div style="font-size:1.8rem; font-weight:bold; color:#6b21a8;"><span id="statWork">0</span></div>
                    <div style="font-size:0.8rem; color:#475569;">⚡ В работе</div>
                </div>
                <div style="background:linear-gradient(135deg, #dcfce7, #86efac); border-radius:0.75rem; padding:0.75rem; text-align:center; box-shadow:0 2px 8px rgba(22,163,74,0.1);">
                    <div style="font-size:1.8rem; font-weight:bold; color:#15803d;"><span id="statDone">0</span></div>
                    <div style="font-size:0.8rem; color:#475569;">✅ Выполнено</div>
                </div>
                <div style="background:linear-gradient(135deg, #fee2e2, #fca5a5); border-radius:0.75rem; padding:0.75rem; text-align:center; box-shadow:0 2px 8px rgba(239,68,68,0.1);">
                    <div style="font-size:1.8rem; font-weight:bold; color:#b91c1c;"><span id="statCancelled">0</span></div>
                    <div style="font-size:0.8rem; color:#475569;">❌ Отменено</div>
                </div>
            </div>
            <div style="display:flex; flex-wrap:wrap; gap:0.5rem; margin-bottom:1rem; background:#f8fafc; padding:0.75rem 1rem; border-radius:0.5rem; border:1px solid #e5e7eb; align-items:center;">
                <div style="flex:1; min-width:180px;"><input type="text" id="search" placeholder="🔍 Поиск по имени..." style="width:100%; padding:0.4rem 0.75rem; border:1px solid #d1d5db; border-radius:0.5rem; outline:none;"></div>
                <select id="statusFilter" style="padding:0.4rem 0.75rem; border:1px solid #d1d5db; border-radius:0.5rem; background:white; outline:none; color:#1f2937;">
                    <option value="все">Все статусы</option>
                    <option value="новая">🟡 Новая</option>
                    <option value="в работе">🟣 В работе</option>
                    <option value="выполнено">✅ Выполнено</option>
                    <option value="отменено">❌ Отменено</option>
                </select>
                <select id="workerFilter" style="padding:0.4rem 0.75rem; border:1px solid #d1d5db; border-radius:0.5rem; background:white; outline:none; color:#1f2937;">
                    <option value="все">👤 Все исполнители</option>
                </select>
                <button id="filterBtn" style="background:#166534; color:white; border:none; border-radius:0.5rem; padding:0.4rem 1rem; cursor:pointer; font-weight:bold;">🔍 Применить</button>
                <button id="createCsvBtn" style="background:#2563eb; color:white; border:none; border-radius:0.5rem; padding:0.4rem 1rem; cursor:pointer; font-weight:bold;">📥 CSV</button>
                <button id="printPdfBtn" style="background:#7c3aed; color:white; border:none; border-radius:0.5rem; padding:0.4rem 1rem; cursor:pointer; font-weight:bold;">🖨️ Печать</button>
            </div>
            <div style="background:white; border-radius:0.75rem; overflow:auto; box-shadow:0 4px 12px rgba(0,0,0,0.05); border:1px solid #e5e7eb;">
                <div style="overflow-x:auto; min-width:1200px;">
                    <div id="requestsList" style="min-height:200px;"></div>
                </div>
                <div style="display:flex; justify-content:space-between; align-items:center; padding:0.5rem 1rem; background:#f8fafc; border-top:1px solid #e5e7eb; flex-wrap:wrap;">
                    <span style="color:#6b7280; font-size:0.85rem;" id="counter">📅 Сегодня создано: 0</span>
                </div>
            </div>
        </div>
        <!-- Вкладка Ресурсы -->
        <div id="tab-resources" class="tab-content hidden">
            <div style="display:flex; justify-content:space-between; align-items:center; margin-bottom:1.5rem; flex-wrap:wrap;">
                <h2 style="font-size:1.5rem; color:#166534; margin:0;">📦 Управление ресурсами</h2>
            </div>
            <div class="flex flex-wrap gap-2 mb-2" style="display:flex; flex-wrap:wrap; gap:0.5rem; margin-bottom:1rem; background:#f8fafc; padding:0.75rem 1rem; border-radius:0.5rem; border:1px solid #e5e7eb;">
                <input type="text" id="resName" placeholder="Название" class="border p-1 rounded" style="flex:1; min-width:150px; padding:0.4rem 0.75rem; border:1px solid #d1d5db; border-radius:0.5rem; outline:none;">
                <input type="text" id="resType" placeholder="Тип" class="border p-1 rounded" style="flex:1; min-width:120px; padding:0.4rem 0.75rem; border:1px solid #d1d5db; border-radius:0.5rem; outline:none;">
                <input type="number" id="resQty" placeholder="Кол-во" class="border p-1 rounded w-24" style="width:100px; padding:0.4rem 0.75rem; border:1px solid #d1d5db; border-radius:0.5rem; outline:none;">
                <button id="addResourceBtn" class="btn btn-blue" style="background:#166534; color:white; border:none; border-radius:0.5rem; padding:0.4rem 1.2rem; cursor:pointer; font-weight:bold;">➕ Добавить</button>
            </div>
            <div class="overflow-x-auto" style="min-width:800px;">
                <table style="width:100%; border-collapse:collapse; background:white; border-radius:0.5rem; overflow:hidden; box-shadow:0 2px 8px rgba(0,0,0,0.05); border:1px solid #e5e7eb;">
                    <thead>
                        <tr style="background:linear-gradient(135deg, #166534, #15803d);">
                            <th style="padding:0.6rem 1rem; text-align:left; color:#ffffff; font-weight:600; white-space:nowrap;">Название</th>
                            <th style="padding:0.6rem 1rem; text-align:left; color:#ffffff; font-weight:600; white-space:nowrap;">Тип</th>
                            <th style="padding:0.6rem 1rem; text-align:left; color:#ffffff; font-weight:600; white-space:nowrap;">Кол-во</th>
                            <th style="padding:0.6rem 1rem; text-align:left; color:#ffffff; font-weight:600; white-space:nowrap;">Действия</th>
                        </tr>
                    </thead>
                    <tbody id="resourcesTableBody"></tbody>
                </table>
            </div>
        </div>
        <!-- Вкладка Сезоны -->
        <div id="tab-seasons" class="tab-content hidden">
            <div style="display:flex; justify-content:space-between; align-items:center; margin-bottom:1.5rem; flex-wrap:wrap;">
                <h2 style="font-size:1.5rem; color:#166534; margin:0;">🌿 Управление сезонами</h2>
            </div>
            <div class="flex flex-wrap gap-2 mb-2" style="display:flex; flex-wrap:wrap; gap:0.5rem; margin-bottom:1rem; background:#f8fafc; padding:0.75rem 1rem; border-radius:0.5rem; border:1px solid #e5e7eb;">
                <input type="text" id="seasonName" placeholder="Название сезона" class="border p-1 rounded" style="flex:1; min-width:150px; padding:0.4rem 0.75rem; border:1px solid #d1d5db; border-radius:0.5rem; outline:none;">
                <input type="text" id="seasonPeriod" placeholder="Период" class="border p-1 rounded" style="flex:1; min-width:120px; padding:0.4rem 0.75rem; border:1px solid #d1d5db; border-radius:0.5rem; outline:none;">
                <input type="text" id="seasonRec" placeholder="Рекомендации" class="border p-1 rounded flex-1" style="flex:2; min-width:180px; padding:0.4rem 0.75rem; border:1px solid #d1d5db; border-radius:0.5rem; outline:none;">
                <button id="addSeasonBtn" class="btn btn-blue" style="background:#166534; color:white; border:none; border-radius:0.5rem; padding:0.4rem 1.2rem; cursor:pointer; font-weight:bold;">➕ Добавить</button>
            </div>
            <div class="overflow-x-auto" style="min-width:800px;">
                <table style="width:100%; border-collapse:collapse; background:white; border-radius:0.5rem; overflow:hidden; box-shadow:0 2px 8px rgba(0,0,0,0.05); border:1px solid #e5e7eb;">
                    <thead>
                        <tr style="background:linear-gradient(135deg, #166534, #15803d);">
                            <th style="padding:0.6rem 1rem; text-align:left; color:#ffffff; font-weight:600; white-space:nowrap;">Название</th>
                            <th style="padding:0.6rem 1rem; text-align:left; color:#ffffff; font-weight:600; white-space:nowrap;">Период</th>
                            <th style="padding:0.6rem 1rem; text-align:left; color:#ffffff; font-weight:600; white-space:nowrap;">Рекомендации</th>
                            <th style="padding:0.6rem 1rem; text-align:left; color:#ffffff; font-weight:600; white-space:nowrap;">Действия</th>
                        </tr>
                    </thead>
                    <tbody id="seasonsTableBody"></tbody>
                </table>
            </div>
        </div>
        <div style="height:3px; background: linear-gradient(to right, #166534, #22c55e, #86efac, #22c55e, #166534); border-radius:4px; margin-top:1.5rem;"></div>
    </div>
    <script>
    function escapeHtml(s){if(!s)return '';return s.replace(/[&<>]/g,m=>({'&':'&amp;','<':'&lt;','>':'&gt;'}[m]));}
    async function loadWorkersList(){
        const res=await fetch('/api/workers');
        const data=await res.json();
        const select=document.getElementById('workerFilter');
        select.innerHTML='<option value="все">👤 Все исполнители</option>';
        data.forEach(w=>{
            const option=document.createElement('option');
            option.value=w.full_name;
            option.textContent=w.full_name;
            select.appendChild(option);
        });
    }
    async function loadResources(){
        const res=await fetch('/api/resources');
        const data=await res.json();
        const tbody=document.getElementById('resourcesTableBody');
        if(!tbody)return;
        tbody.innerHTML='';
        for(let r of data){
            const row=tbody.insertRow();
            row.style.borderBottom='1px solid #e5e7eb';
            row.style.transition='background 0.2s';
            row.onmouseover=()=>row.style.background='#f8fafc';
            row.onmouseout=()=>row.style.background='transparent';
            const cell1=row.insertCell(0); cell1.style.padding='0.6rem 1rem'; cell1.style.color='#1a1a1a'; cell1.style.fontWeight='500'; cell1.style.whiteSpace='nowrap'; cell1.innerHTML=escapeHtml(r.name);
            const cell2=row.insertCell(1); cell2.style.padding='0.6rem 1rem'; cell2.style.color='#1a1a1a'; cell2.style.whiteSpace='nowrap'; cell2.innerHTML=escapeHtml(r.type);
            const cell3=row.insertCell(2); cell3.style.padding='0.6rem 1rem'; cell3.style.color='#1a1a1a'; cell3.style.fontWeight='600'; cell3.style.whiteSpace='nowrap'; cell3.innerHTML=r.quantity;
            const cell4=row.insertCell(3); cell4.style.padding='0.6rem 1rem'; cell4.style.whiteSpace='nowrap'; cell4.innerHTML=`<button onclick="editResource(${r.id},${r.quantity})" style="background:#eab308; color:white; border:none; border-radius:0.25rem; padding:0.2rem 0.6rem; font-size:0.75rem; cursor:pointer;">✏️ Изменить</button> <button onclick="delResource(${r.id})" style="background:#ef4444; color:white; border:none; border-radius:0.25rem; padding:0.2rem 0.6rem; font-size:0.75rem; cursor:pointer;">🗑️ Удалить</button>`;
        }
    }
    window.editResource=async(id,currentQty)=>{
        let newQty=prompt('Новое количество:',currentQty);
        if(newQty!==null&&!isNaN(parseInt(newQty))){
            await fetch('/api/resource/'+id,{method:'PUT',headers:{'Content-Type':'application/json'},body:JSON.stringify({quantity:parseInt(newQty)})});
            loadResources();
        }
    };
    window.delResource=async(id)=>{
        if(confirm('Удалить ресурс?')) await fetch('/api/resource/'+id,{method:'DELETE'});
        loadResources();
    };
    document.getElementById('addResourceBtn')?.addEventListener('click',async()=>{
        const name=document.getElementById('resName').value;
        const type=document.getElementById('resType').value;
        const qty=document.getElementById('resQty').value;
        if(name&&type&&qty){
            await fetch('/api/resource',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({name,type,quantity:qty})});
            loadResources();
            document.getElementById('resName').value='';
            document.getElementById('resType').value='';
            document.getElementById('resQty').value='';
        }
    });
    async function loadSeasons(){
        const res=await fetch('/api/seasons');
        const data=await res.json();
        const tbody=document.getElementById('seasonsTableBody');
        if(!tbody)return;
        tbody.innerHTML='';
        for(let s of data){
            const row=tbody.insertRow();
            row.style.borderBottom='1px solid #e5e7eb';
            row.style.transition='background 0.2s';
            row.onmouseover=()=>row.style.background='#f8fafc';
            row.onmouseout=()=>row.style.background='transparent';
            const cell1=row.insertCell(0); cell1.style.padding='0.6rem 1rem'; cell1.style.color='#1a1a1a'; cell1.style.fontWeight='500'; cell1.style.whiteSpace='nowrap'; cell1.innerHTML=escapeHtml(s.name);
            const cell2=row.insertCell(1); cell2.style.padding='0.6rem 1rem'; cell2.style.color='#1a1a1a'; cell2.style.whiteSpace='nowrap'; cell2.innerHTML=escapeHtml(s.period);
            const cell3=row.insertCell(2); cell3.style.padding='0.6rem 1rem'; cell3.style.color='#1a1a1a'; cell3.style.whiteSpace='nowrap'; cell3.innerHTML=escapeHtml(s.recommendations);
            const cell4=row.insertCell(3); cell4.style.padding='0.6rem 1rem'; cell4.style.whiteSpace='nowrap'; cell4.innerHTML=`<button onclick="editSeason(${s.id},'${escapeHtml(s.recommendations)}')" style="background:#eab308; color:white; border:none; border-radius:0.25rem; padding:0.2rem 0.6rem; font-size:0.75rem; cursor:pointer;">✏️ Ред.</button> <button onclick="delSeason(${s.id})" style="background:#ef4444; color:white; border:none; border-radius:0.25rem; padding:0.2rem 0.6rem; font-size:0.75rem; cursor:pointer;">🗑️ Удалить</button>`;
        }
    }
    window.editSeason=async(id,oldRec)=>{
        let newRec=prompt('Новые рекомендации:',oldRec);
        if(newRec!==null){
            await fetch('/api/season/'+id,{method:'PUT',headers:{'Content-Type':'application/json'},body:JSON.stringify({recommendations:newRec})});
            loadSeasons();
        }
    };
    window.delSeason=async(id)=>{
        if(confirm('Удалить сезон?')) await fetch('/api/season/'+id,{method:'DELETE'});
        loadSeasons();
    };
    document.getElementById('addSeasonBtn')?.addEventListener('click',async()=>{
        const name=document.getElementById('seasonName').value;
        const period=document.getElementById('seasonPeriod').value;
        const rec=document.getElementById('seasonRec').value;
        if(name&&period&&rec){
            await fetch('/api/season',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({name,period,recommendations:rec})});
            loadSeasons();
            document.getElementById('seasonName').value='';
            document.getElementById('seasonPeriod').value='';
            document.getElementById('seasonRec').value='';
        }
    });
    async function loadStats(){
        const res=await fetch('/api/stats');
        const stats=await res.json();
        document.getElementById('statTotal').innerText=stats.total;
        document.getElementById('statNew').innerText=stats.new;
        document.getElementById('statWork').innerText=stats.work;
        document.getElementById('statDone').innerText=stats.done;
        document.getElementById('statCancelled').innerText=stats.cancelled;
    }
    async function loadRequests(){
        const search=document.getElementById('search').value;
        const status=document.getElementById('statusFilter').value;
        const worker=document.getElementById('workerFilter').value;
        const url=`/api/requests?search=${encodeURIComponent(search)}&status=${encodeURIComponent(status)}&worker=${encodeURIComponent(worker)}`;
        const res=await fetch(url);
        const data=await res.json();
        const container=document.getElementById('requestsList');
        if(!data.requests||data.requests.length===0){
            container.innerHTML='<div style="text-align:center; padding:2rem; color:#6b7280;">📭 Нет заявок</div>';
            return;
        }
        let html=`<table style="width:100%; border-collapse:collapse; min-width:1200px;">
            <thead>
                <tr style="background:linear-gradient(135deg, #166534, #15803d);">
                    <th style="padding:0.6rem 1rem; text-align:left; color:#ffffff; font-weight:600; font-size:0.85rem; white-space:nowrap;">ID</th>
                    <th style="padding:0.6rem 1rem; text-align:left; color:#ffffff; font-weight:600; font-size:0.85rem; white-space:nowrap;">Клиент</th>
                    <th style="padding:0.6rem 1rem; text-align:left; color:#ffffff; font-weight:600; font-size:0.85rem; white-space:nowrap;">Телефон</th>
                    <th style="padding:0.6rem 1rem; text-align:left; color:#ffffff; font-weight:600; font-size:0.85rem; white-space:nowrap;">Услуга</th>
                    <th style="padding:0.6rem 1rem; text-align:left; color:#ffffff; font-weight:600; font-size:0.85rem; white-space:nowrap;">Статус</th>
                    <th style="padding:0.6rem 1rem; text-align:left; color:#ffffff; font-weight:600; font-size:0.85rem; white-space:nowrap;">Исполнитель</th>
                    <th style="padding:0.6rem 1rem; text-align:left; color:#ffffff; font-weight:600; font-size:0.85rem; white-space:nowrap;">Дата создания</th>
                    <th style="padding:0.6rem 1rem; text-align:center; color:#ffffff; font-weight:600; font-size:0.85rem; white-space:nowrap;">Действия</th>
                </tr>
            </thead>
            <tbody>`;
        for(let r of data.requests){
            let workerSelect = '<select onchange="assignWorker('+r.id+',this.value)" style="padding:0.2rem 0.4rem; border:1px solid #d1d5db; border-radius:0.25rem; background:white; font-size:0.8rem; color:#1a1a1a; min-width:100px;">';
            workerSelect += '<option value="" style="color:#1a1a1a;">Не назначен</option>';
            if(r.workers_list){
                for(let w of r.workers_list){
                    let selected = (r.assigned_worker_id == w.id) ? 'selected' : '';
                    workerSelect += `<option value="${w.id}" ${selected} style="color:#1a1a1a;">${escapeHtml(w.full_name)}</option>`;
                }
            }
            workerSelect += '</select>';

            let statusBadge = '';
            if(r.status === 'новая') statusBadge = `<span style="background:#fef9c3; color:#854d0e; padding:0.2rem 0.6rem; border-radius:1rem; font-size:0.75rem; font-weight:bold; white-space:nowrap;">🟡 новая</span>`;
            else if(r.status === 'в работе') statusBadge = `<span style="background:#f3e8ff; color:#6b21a8; padding:0.2rem 0.6rem; border-radius:1rem; font-size:0.75rem; font-weight:bold; white-space:nowrap;">🟣 в работе</span>`;
            else if(r.status === 'выполнено') statusBadge = `<span style="background:#dcfce7; color:#15803d; padding:0.2rem 0.6rem; border-radius:1rem; font-size:0.75rem; font-weight:bold; white-space:nowrap;">✅ выполнено</span>`;
            else if(r.status === 'отменено') statusBadge = `<span style="background:#fee2e2; color:#b91c1c; padding:0.2rem 0.6rem; border-radius:1rem; font-size:0.75rem; font-weight:bold; white-space:nowrap;">❌ отменено</span>`;

            html+=`<tr style="border-bottom:1px solid #e5e7eb; transition:background 0.2s;">
                <td style="padding:0.6rem 1rem; font-weight:bold; color:#166534; font-size:0.9rem; white-space:nowrap;">${r.id}</td>
                <td style="padding:0.6rem 1rem; color:#1a1a1a; font-weight:500; white-space:nowrap;">${escapeHtml(r.customer_name)}</td>
                <td style="padding:0.6rem 1rem; color:#1a1a1a; white-space:nowrap;">${escapeHtml(r.phone)}</td>
                <td style="padding:0.6rem 1rem; color:#1a1a1a; white-space:nowrap;">${escapeHtml(r.service_type)}</td>
                <td style="padding:0.6rem 1rem; white-space:nowrap;">${statusBadge}</td>
                <td style="padding:0.6rem 1rem; white-space:nowrap;">${workerSelect}</td>
                <td style="padding:0.6rem 1rem; color:#1a1a1a; font-size:0.85rem; white-space:nowrap;">${r.created_at}</td>
                <td style="padding:0.6rem 1rem; text-align:center; white-space:nowrap;">
                    <button onclick="changeStatus(${r.id},'в работе')" style="background:#2563eb; color:white; border:none; border-radius:0.25rem; padding:0.2rem 0.5rem; font-size:0.7rem; cursor:pointer; white-space:nowrap;">▶ В работу</button>
                    <button onclick="changeStatus(${r.id},'выполнено')" style="background:#16a34a; color:white; border:none; border-radius:0.25rem; padding:0.2rem 0.5rem; font-size:0.7rem; cursor:pointer; white-space:nowrap;">✓ Выполнить</button>
                    <button onclick="changeStatus(${r.id},'отменено')" style="background:#ef4444; color:white; border:none; border-radius:0.25rem; padding:0.2rem 0.5rem; font-size:0.7rem; cursor:pointer; white-space:nowrap;">✗ Отменить</button>
                    <button onclick="deleteRequest(${r.id})" style="background:#6b7280; color:white; border:none; border-radius:0.25rem; padding:0.2rem 0.5rem; font-size:0.7rem; cursor:pointer; white-space:nowrap;">🗑 Удалить</button>
                </td>
            </tr>`;
        }
        html+='</tbody></table>';
        container.innerHTML=html;
        document.getElementById('counter').innerHTML=`📅 Сегодня создано: ${data.today_count}`;
    }
    window.assignWorker=async(id,workerId)=>{
        if(!workerId)workerId=null;
        await fetch(`/api/request/${id}`,{method:'PUT',headers:{'Content-Type':'application/json'},body:JSON.stringify({assigned_worker_id:workerId})});
        loadRequests();
        loadStats();
    };
    window.changeStatus=async(id,status)=>{
        await fetch(`/api/request/${id}`,{method:'PUT',headers:{'Content-Type':'application/json'},body:JSON.stringify({status})});
        loadRequests();
        loadStats();
    };
    window.deleteRequest=async(id)=>{
        if(confirm('Удалить заявку навсегда?')){
            await fetch(`/api/request/${id}`,{method:'DELETE'});
            loadRequests();
            loadStats();
        }
    };
    document.getElementById('filterBtn')?.addEventListener('click',loadRequests);
    document.getElementById('createCsvBtn')?.addEventListener('click',()=>window.location.href='/api/report/csv');
    document.getElementById('printPdfBtn')?.addEventListener('click',()=>window.open('/admin_report_print','_blank'));
    loadWorkersList();
    loadResources();
    loadSeasons();
    loadStats();
    loadRequests();
    setInterval(()=>{loadStats();loadRequests();},15000);
    </script>
    '''

    WORKER_PANEL_HTML = '''
    <div style="position:relative; padding:0 0.5rem;">
        <div style="height:4px; background: linear-gradient(to right, #166534, #22c55e, #86efac, #22c55e, #166534); border-radius:4px; margin-bottom:1.5rem;"></div>
        <div class="card">
            <h2 style="font-size:1.5rem; color:#166534; margin:0 0 0.25rem 0;">🔨 Мои задачи</h2>
            <p style="color:#6b7280; margin:0 0 1rem 0; font-size:0.9rem;">Список задач, назначенных вам</p>
            <div style="display:grid; grid-template-columns:repeat(3,1fr); gap:0.75rem; margin-bottom:1.5rem;">
                <div style="background:linear-gradient(135deg, #eff6ff, #dbeafe); border-radius:0.75rem; padding:0.75rem; text-align:center; box-shadow:0 2px 8px rgba(59,130,246,0.1);">
                    <div style="font-size:1.8rem; font-weight:bold; color:#1e40af;" id="workerStatTotal">0</div>
                    <div style="font-size:0.8rem; color:#475569;">📋 Всего</div>
                </div>
                <div style="background:linear-gradient(135deg, #f3e8ff, #e9d5ff); border-radius:0.75rem; padding:0.75rem; text-align:center; box-shadow:0 2px 8px rgba(147,51,234,0.1);">
                    <div style="font-size:1.8rem; font-weight:bold; color:#6b21a8;" id="workerStatWork">0</div>
                    <div style="font-size:0.8rem; color:#475569;">⚡ В работе</div>
                </div>
                <div style="background:linear-gradient(135deg, #dcfce7, #86efac); border-radius:0.75rem; padding:0.75rem; text-align:center; box-shadow:0 2px 8px rgba(22,163,74,0.1);">
                    <div style="font-size:1.8rem; font-weight:bold; color:#15803d;" id="workerStatDone">0</div>
                    <div style="font-size:0.8rem; color:#475569;">✅ Выполнено</div>
                </div>
            </div>
            <div style="display:flex; flex-wrap:wrap; gap:0.5rem; margin-bottom:1rem; background:#f8fafc; padding:0.75rem 1rem; border-radius:0.5rem; border:1px solid #e5e7eb; align-items:center;">
                <label style="font-weight:500; color:#1f2937; margin-right:0.5rem;">👤 Работник:</label>
                <select id="workerSelect" style="flex:1; min-width:200px; padding:0.4rem 0.75rem; border:1px solid #d1d5db; border-radius:0.5rem; background:white; outline:none; color:#1f2937;"></select>
                <button id="loadWorkerTasks" style="background:#166534; color:white; border:none; border-radius:0.5rem; padding:0.4rem 1rem; cursor:pointer; font-weight:bold;">📋 Загрузить</button>
            </div>
            <div style="background:white; border-radius:0.75rem; overflow:auto; box-shadow:0 4px 12px rgba(0,0,0,0.05); border:1px solid #e5e7eb;">
                <div style="overflow-x:auto; min-width:700px;">
                    <div id="workerTasksList" style="min-height:150px;"></div>
                </div>
            </div>
        </div>
        <div style="height:3px; background: linear-gradient(to right, #166534, #22c55e, #86efac, #22c55e, #166534); border-radius:4px; margin-top:1.5rem;"></div>
    </div>
    <script>
    function escapeHtml(s){if(!s)return '';return s.replace(/[&<>]/g,m=>({'&':'&amp;','<':'&lt;','>':'&gt;'}[m]));}
    async function loadWorkersList(){
        const res=await fetch('/api/workers');
        const data=await res.json();
        const select=document.getElementById('workerSelect');
        select.innerHTML='';
        data.forEach(w=>{
            const option=document.createElement('option');
            option.value=w.id;
            option.textContent=w.full_name;
            select.appendChild(option);
        });
        if(data.length > 0) { loadTasks(); }
    }
    async function loadTasks(){
        const workerId=document.getElementById('workerSelect').value;
        if(!workerId) { document.getElementById('workerTasksList').innerHTML='<div style="text-align:center; padding:2rem; color:#6b7280;">Выберите работника</div>'; return; }
        const res=await fetch(`/api/worker_tasks?worker_id=${workerId}`);
        const data=await res.json();
        const container=document.getElementById('workerTasksList');
        if(!data.tasks||data.tasks.length===0){ container.innerHTML='<div style="text-align:center; padding:2rem; color:#6b7280;">📭 Нет задач</div>'; document.getElementById('workerStatTotal').innerText='0'; document.getElementById('workerStatWork').innerText='0'; document.getElementById('workerStatDone').innerText='0'; return; }
        let total = data.tasks.length; let work = data.tasks.filter(t => t.status === 'в работе').length; let done = data.tasks.filter(t => t.status === 'выполнено').length;
        document.getElementById('workerStatTotal').innerText=total; document.getElementById('workerStatWork').innerText=work; document.getElementById('workerStatDone').innerText=done;
        let html=`<table style="width:100%; border-collapse:collapse;">
            <thead><tr style="background:linear-gradient(135deg, #166534, #15803d);">
                <th style="padding:0.6rem 1rem; text-align:left; color:#ffffff; font-weight:600; font-size:0.85rem; white-space:nowrap;">ID</th>
                <th style="padding:0.6rem 1rem; text-align:left; color:#ffffff; font-weight:600; font-size:0.85rem; white-space:nowrap;">Клиент</th>
                <th style="padding:0.6rem 1rem; text-align:left; color:#ffffff; font-weight:600; font-size:0.85rem; white-space:nowrap;">Услуга</th>
                <th style="padding:0.6rem 1rem; text-align:left; color:#ffffff; font-weight:600; font-size:0.85rem; white-space:nowrap;">Статус</th>
                <th style="padding:0.6rem 1rem; text-align:center; color:#ffffff; font-weight:600; font-size:0.85rem; white-space:nowrap;">Действие</th>
            </tr></thead><tbody>`;
        for(let t of data.tasks){
            let statusBadge = '';
            if(t.status === 'новая') statusBadge = `<span style="background:#fef9c3; color:#854d0e; padding:0.2rem 0.6rem; border-radius:1rem; font-size:0.75rem; font-weight:bold; white-space:nowrap;">🟡 новая</span>`;
            else if(t.status === 'в работе') statusBadge = `<span style="background:#f3e8ff; color:#6b21a8; padding:0.2rem 0.6rem; border-radius:1rem; font-size:0.75rem; font-weight:bold; white-space:nowrap;">🟣 в работе</span>`;
            else if(t.status === 'выполнено') statusBadge = `<span style="background:#dcfce7; color:#15803d; padding:0.2rem 0.6rem; border-radius:1rem; font-size:0.75rem; font-weight:bold; white-space:nowrap;">✅ выполнено</span>`;
            else if(t.status === 'отменено') statusBadge = `<span style="background:#fee2e2; color:#b91c1c; padding:0.2rem 0.6rem; border-radius:1rem; font-size:0.75rem; font-weight:bold; white-space:nowrap;">❌ отменено</span>`;
            let actionBtn = (t.status === 'в работе') ? `<button onclick="completeTask(${t.id})" style="background:#16a34a; color:white; border:none; border-radius:0.25rem; padding:0.2rem 0.6rem; font-size:0.75rem; cursor:pointer; white-space:nowrap;">✅ Завершить</button>` : '<span style="color:#6b7280; font-size:0.8rem;">—</span>';
            html+=`<tr style="border-bottom:1px solid #e5e7eb; transition:background 0.2s;">
                <td style="padding:0.6rem 1rem; font-weight:bold; color:#166534; white-space:nowrap;">#${t.id}</td>
                <td style="padding:0.6rem 1rem; color:#1a1a1a; font-weight:500; white-space:nowrap;">${escapeHtml(t.customer_name)}</td>
                <td style="padding:0.6rem 1rem; color:#1a1a1a; white-space:nowrap;">${escapeHtml(t.service_type)}</td>
                <td style="padding:0.6rem 1rem; white-space:nowrap;">${statusBadge}</td>
                <td style="padding:0.6rem 1rem; text-align:center; white-space:nowrap;">${actionBtn}</td>
            </tr>`;
        }
        html+='</tbody></table>';
        container.innerHTML=html;
    }
    window.completeTask=async(id)=>{ if(confirm('Отметить задачу как выполненную?')){ await fetch(`/api/request/${id}`,{method:'PUT',headers:{'Content-Type':'application/json'},body:JSON.stringify({status:'выполнено'})}); loadTasks(); } };
    document.getElementById('loadWorkerTasks')?.addEventListener('click', loadTasks);
    loadWorkersList();
    </script>
    '''

    CLIENT_PANEL_HTML = '''
    <div style="position:relative; padding:0 0.5rem;">
        <div style="height:4px; background: linear-gradient(to right, #166534, #22c55e, #86efac, #22c55e, #166534); border-radius:4px; margin-bottom:1.5rem;"></div>
        <div style="background:white; border-radius:0.75rem; padding:1.5rem; box-shadow:0 4px 12px rgba(0,0,0,0.05); border:1px solid #e5e7eb; margin-bottom:1.5rem;">
            <h2 style="font-size:1.5rem; color:#166534; margin:0 0 0.25rem 0;">✨ Новая заявка</h2>
            <p style="color:#6b7280; margin:0 0 1rem 0; font-size:0.9rem;">Заполните форму, чтобы создать новую заявку</p>
            <form id="requestForm">
                <div style="display:grid; grid-template-columns:repeat(auto-fit, minmax(200px,1fr)); gap:0.75rem; margin-bottom:0.75rem;">
                    <div><label style="display:block; font-weight:500; color:#1f2937; margin-bottom:0.25rem;">Ваше имя *</label><input type="text" id="name" placeholder="Иван Иванов" required style="width:100%; padding:0.5rem; border:1px solid #d1d5db; border-radius:0.5rem; outline:none;"></div>
                    <div><label style="display:block; font-weight:500; color:#1f2937; margin-bottom:0.25rem;">Телефон *</label><input type="tel" id="phone" placeholder="+7(900)111-22-33" required pattern="[\\+0-9\\-\\(\\)\\s]+" style="width:100%; padding:0.5rem; border:1px solid #d1d5db; border-radius:0.5rem; outline:none;"></div>
                    <div><label style="display:block; font-weight:500; color:#1f2937; margin-bottom:0.25rem;">Адрес</label><input type="text" id="address" placeholder="г. Волгодонск, ул. ..." style="width:100%; padding:0.5rem; border:1px solid #d1d5db; border-radius:0.5rem; outline:none;"></div>
                    <div><label style="display:block; font-weight:500; color:#1f2937; margin-bottom:0.25rem;">Услуга</label><select id="service" style="width:100%; padding:0.5rem; border:1px solid #d1d5db; border-radius:0.5rem; outline:none; background:white; color:#1f2937;"><option value="озеленение">Озеленение</option><option value="посадка">Посадка</option><option value="уход">Уход</option><option value="дизайн">Дизайн</option></select></div>
                </div>
                <div style="margin-bottom:0.75rem;"><label style="display:block; font-weight:500; color:#1f2937; margin-bottom:0.25rem;">Описание</label><textarea id="desc" rows="3" placeholder="Опишите ваши пожелания..." style="width:100%; padding:0.5rem; border:1px solid #d1d5db; border-radius:0.5rem; outline:none; resize:vertical; min-height:80px;"></textarea></div>
                <button type="submit" style="background:#166534; color:white; border:none; border-radius:0.5rem; padding:0.6rem 1.5rem; font-weight:bold; cursor:pointer;">📨 Отправить заявку</button>
                <div id="msg" class="mt-2 text-center" style="margin-top:0.5rem; font-weight:500;"></div>
            </form>
        </div>
        <div style="background:white; border-radius:0.75rem; padding:1.5rem; box-shadow:0 4px 12px rgba(0,0,0,0.05); border:1px solid #e5e7eb;">
            <h2 style="font-size:1.5rem; color:#166534; margin:0 0 0.25rem 0;">📋 Мои заявки</h2>
            <p style="color:#6b7280; margin:0 0 1rem 0; font-size:0.9rem;">Поиск и просмотр ваших заявок</p>
            <div style="display:flex; flex-wrap:wrap; gap:0.5rem; margin-bottom:1rem; background:#f8fafc; padding:0.75rem 1rem; border-radius:0.5rem; border:1px solid #e5e7eb; align-items:center;">
                <label style="font-weight:500; color:#1f2937; margin-right:0.5rem;">👤 Ваше имя:</label>
                <input type="text" id="clientSearch" placeholder="Иванов Иван" style="flex:1; min-width:200px; padding:0.4rem 0.75rem; border:1px solid #d1d5db; border-radius:0.5rem; outline:none;">
                <button id="searchClientBtn" style="background:#166534; color:white; border:none; border-radius:0.5rem; padding:0.4rem 1rem; cursor:pointer; font-weight:bold;">🔍 Показать</button>
            </div>
            <div style="overflow-x:auto; min-width:700px;"><div id="clientRequestsList" style="min-height:100px;"></div></div>
        </div>
        <div style="height:3px; background: linear-gradient(to right, #166534, #22c55e, #86efac, #22c55e, #166534); border-radius:4px; margin-top:1.5rem;"></div>
    </div>
    <script>
    function escapeHtml(s){if(!s)return '';return s.replace(/[&<>]/g,m=>({'&':'&amp;','<':'&lt;','>':'&gt;'}[m]));}
    async function loadClientRequests(){
        const name=document.getElementById('clientSearch').value.trim();
        if(!name){ document.getElementById('clientRequestsList').innerHTML='<div style="text-align:center; padding:2rem; color:#6b7280;">Введите ваше имя для поиска</div>'; return; }
        const res=await fetch(`/api/client_requests?name=${encodeURIComponent(name)}`);
        const data=await res.json();
        if(data.error){ document.getElementById('clientRequestsList').innerHTML='<div style="text-align:center; padding:2rem; color:#ef4444;">Ошибка загрузки</div>'; return; }
        if(!data.requests || data.requests.length===0){ document.getElementById('clientRequestsList').innerHTML='<div style="text-align:center; padding:2rem; color:#6b7280;">📭 Заявок не найдено</div>'; return; }
        let html=`<table style="width:100%; border-collapse:collapse;">
            <thead><tr style="background:linear-gradient(135deg, #166534, #15803d);">
                <th style="padding:0.6rem 1rem; text-align:left; color:#ffffff; font-weight:600; font-size:0.85rem; white-space:nowrap;">ID</th>
                <th style="padding:0.6rem 1rem; text-align:left; color:#ffffff; font-weight:600; font-size:0.85rem; white-space:nowrap;">Услуга</th>
                <th style="padding:0.6rem 1rem; text-align:left; color:#ffffff; font-weight:600; font-size:0.85rem; white-space:nowrap;">Статус</th>
                <th style="padding:0.6rem 1rem; text-align:left; color:#ffffff; font-weight:600; font-size:0.85rem; white-space:nowrap;">Создана</th>
                <th style="padding:0.6rem 1rem; text-align:left; color:#ffffff; font-weight:600; font-size:0.85rem; white-space:nowrap;">Описание</th>
            </tr></thead><tbody>`;
        data.requests.forEach(r=>{
            let statusBadge = '';
            if(r.status === 'новая') statusBadge = `<span style="background:#fef9c3; color:#854d0e; padding:0.2rem 0.6rem; border-radius:1rem; font-size:0.75rem; font-weight:bold; white-space:nowrap;">🟡 новая</span>`;
            else if(r.status === 'в работе') statusBadge = `<span style="background:#f3e8ff; color:#6b21a8; padding:0.2rem 0.6rem; border-radius:1rem; font-size:0.75rem; font-weight:bold; white-space:nowrap;">🟣 в работе</span>`;
            else if(r.status === 'выполнено') statusBadge = `<span style="background:#dcfce7; color:#15803d; padding:0.2rem 0.6rem; border-radius:1rem; font-size:0.75rem; font-weight:bold; white-space:nowrap;">✅ выполнено</span>`;
            else if(r.status === 'отменено') statusBadge = `<span style="background:#fee2e2; color:#b91c1c; padding:0.2rem 0.6rem; border-radius:1rem; font-size:0.75rem; font-weight:bold; white-space:nowrap;">❌ отменено</span>`;
            html+=`<tr style="border-bottom:1px solid #e5e7eb; transition:background 0.2s;">
                <td style="padding:0.6rem 1rem; font-weight:bold; color:#166534; white-space:nowrap;">#${r.id}</td>
                <td style="padding:0.6rem 1rem; color:#1a1a1a; white-space:nowrap;">${escapeHtml(r.service_type)}</td>
                <td style="padding:0.6rem 1rem; white-space:nowrap;">${statusBadge}</td>
                <td style="padding:0.6rem 1rem; color:#1a1a1a; white-space:nowrap;">${r.created_at}</td>
                <td style="padding:0.6rem 1rem; color:#1a1a1a;">${escapeHtml(r.description||'')}</td>
            </tr>`;
        });
        html+='</tbody></table>';
        document.getElementById('clientRequestsList').innerHTML=html;
    }
    document.getElementById('requestForm').addEventListener('submit', async (e)=>{
        e.preventDefault();
        const name=document.getElementById('name').value;
        const phone=document.getElementById('phone').value;
        const address=document.getElementById('address').value;
        const service=document.getElementById('service').value;
        const desc=document.getElementById('desc').value;
        const res=await fetch('/api/request',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({name, phone, address, service_type:service, description:desc})});
        const result=await res.json();
        const msgDiv=document.getElementById('msg');
        if(res.ok){ msgDiv.innerHTML='<span style="color:#15803d;">✅ Заявка успешно создана!</span>'; document.getElementById('requestForm').reset(); const searchName=document.getElementById('clientSearch').value.trim(); if(searchName) loadClientRequests(); }
        else { msgDiv.innerHTML='<span style="color:#dc2626;">❌ Ошибка: '+result.message+'</span>'; }
    });
    document.getElementById('searchClientBtn').addEventListener('click', loadClientRequests);
    </script>
    '''

    # ---------- ИНИЦИАЛИЗАЦИЯ БАЗЫ ДАННЫХ ----------
    def init_db():
        conn = sqlite3.connect('souzidatel.db')
        c = conn.cursor()
        c.execute('PRAGMA foreign_keys = ON')
        c.execute('''
            CREATE TABLE IF NOT EXISTS users (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                username TEXT UNIQUE NOT NULL,
                password TEXT NOT NULL,
                full_name TEXT NOT NULL,
                role TEXT NOT NULL
            )
        ''')
        c.execute('''
            CREATE TABLE IF NOT EXISTS requests (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                customer_name TEXT NOT NULL,
                phone TEXT NOT NULL,
                address TEXT,
                description TEXT,
                service_type TEXT NOT NULL,
                status TEXT DEFAULT 'новая',
                assigned_worker_id INTEGER REFERENCES users(id) ON DELETE SET NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                started_at TIMESTAMP,
                completed_at TIMESTAMP
            )
        ''')
        try:
            c.execute('ALTER TABLE requests ADD COLUMN assigned_worker_id INTEGER REFERENCES users(id) ON DELETE SET NULL')
        except sqlite3.OperationalError:
            pass
        try:
            c.execute('ALTER TABLE requests DROP COLUMN assigned_worker')
        except sqlite3.OperationalError:
            pass
        c.execute('''
            CREATE TABLE IF NOT EXISTS resources (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                name TEXT NOT NULL,
                type TEXT,
                quantity INTEGER DEFAULT 0
            )
        ''')
        c.execute('''
            CREATE TABLE IF NOT EXISTS seasons (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                name TEXT NOT NULL,
                period TEXT,
                recommendations TEXT
            )
        ''')
        c.execute('CREATE INDEX IF NOT EXISTS idx_requests_status ON requests(status)')
        c.execute('CREATE INDEX IF NOT EXISTS idx_requests_assigned_worker ON requests(assigned_worker_id)')
        c.execute('CREATE INDEX IF NOT EXISTS idx_requests_created_at ON requests(created_at)')
        c.execute('CREATE INDEX IF NOT EXISTS idx_users_username ON users(username)')

        users = [
            ('admin', hashlib.md5('admin123'.encode()).hexdigest(), 'Администратор', 'admin'),
            ('client', hashlib.md5('client123'.encode()).hexdigest(), 'Клиент (Иванов)', 'client'),
            ('worker', hashlib.md5('worker123'.encode()).hexdigest(), 'Исполнитель (Петров)', 'worker'),
            ('anna', hashlib.md5('anna123'.encode()).hexdigest(), 'Анна Сидорова', 'worker'),
            ('ivan_worker', hashlib.md5('ivan123'.encode()).hexdigest(), 'Иван Петров', 'worker'),
            ('elena', hashlib.md5('elena123'.encode()).hexdigest(), 'Елена Васильева', 'client')
        ]
        for u in users:
            c.execute("SELECT id FROM users WHERE username=?", (u[0],))
            if not c.fetchone():
                c.execute("INSERT INTO users (username, password, full_name, role) VALUES (?,?,?,?)", u)

        c.execute("SELECT COUNT(*) FROM requests")
        if c.fetchone()[0] == 0:
            now = datetime.now()
            c.execute('INSERT INTO requests (customer_name, phone, service_type, description, status, assigned_worker_id, created_at) VALUES (?,?,?,?,?,?,?)',
                      ('Иванов Иван', '+7(909)111-22-33', 'озеленение', 'Озеленить двор: посадить газон, кустарники', 'новая',
                       None, now - timedelta(days=2)))
            c.execute('INSERT INTO requests (customer_name, phone, service_type, description, status, assigned_worker_id, created_at, started_at) VALUES (?,?,?,?,?,?,?,?)',
                      ('Петрова Елена', '+7(909)444-55-66', 'посадка', 'Посадка роз и многолетников', 'в работе', None,
                       now - timedelta(days=1), now - timedelta(hours=5)))
            c.execute('INSERT INTO requests (customer_name, phone, service_type, description, status, assigned_worker_id, created_at, started_at, completed_at) VALUES (?,?,?,?,?,?,?,?,?)',
                      ('Василий Петренко', '89381206593', 'озеленение', 'Озеленение двора частного дома', 'выполнено', None,
                       now - timedelta(days=5), now - timedelta(days=4), now - timedelta(days=2)))
            c.execute('INSERT INTO requests (customer_name, phone, service_type, description, status, assigned_worker_id, created_at) VALUES (?,?,?,?,?,?,?)',
                      ('Сергей Михайлов', '+7(918)555-12-34', 'дизайн', 'Разработка ландшафтного дизайна участка 15 соток',
                       'новая', None, now - timedelta(hours=3)))
            c.execute('INSERT INTO requests (customer_name, phone, service_type, description, status, assigned_worker_id, created_at) VALUES (?,?,?,?,?,?,?)',
                      ('Татьяна Смирнова', '+7(928)777-88-99', 'уход',
                       'Ежемесячный уход за садом (стрижка, полив, удобрение)', 'новая', None, now - timedelta(hours=1)))

        c.execute("SELECT COUNT(*) FROM resources")
        if c.fetchone()[0] == 0:
            resources = [('Рассада цветов (петуния, бархатцы)', 'рассада', 500),
                         ('Газонокосилка электрическая', 'техника', 2),
                         ('Удобрение универсальное (10 кг)', 'удобрение', 100),
                         ('Секаторы профессиональные', 'инструмент', 15), ('Грабли веерные', 'инструмент', 20),
                         ('Лопата штыковая', 'инструмент', 25), ('Почва плодородная (литры)', 'материал', 3000)]
            for r in resources:
                c.execute("INSERT INTO resources (name, type, quantity) VALUES (?,?,?)", r)

        c.execute("SELECT COUNT(*) FROM seasons")
        if c.fetchone()[0] == 0:
            seasons = [('Весна', 'март-май', 'Посадка деревьев, удобрение почвы, обрезка кустарников'),
                       ('Лето', 'июнь-август', 'Полив, стрижка газонов, борьба с вредителями'),
                       ('Осень', 'сентябрь-ноябрь', 'Уборка листьев, подготовка к зиме, перекопка почвы'),
                       ('Зима', 'декабрь-февраль', 'Укрытие растений, очистка снега, планирование весенних работ')]
            for s in seasons:
                c.execute("INSERT INTO seasons (name, period, recommendations) VALUES (?,?,?)", s)

        conn.commit()
        conn.close()

    init_db()

    # ------------------- ФУНКЦИЯ db_query -------------------
    def db_query(query, params=(), fetch_one=False, fetch_all=False, commit=False):
        conn = sqlite3.connect('souzidatel.db')
        conn.row_factory = sqlite3.Row
        c = conn.cursor()
        c.execute(query, params)
        result = None
        if fetch_one:
            result = c.fetchone()
        elif fetch_all:
            result = c.fetchall()
        if commit:
            conn.commit()
        conn.close()
        return result

    # ------------------- МАРШРУТЫ -------------------
    @app.route('/')
    def index():
        if not session.get('user_id'):
            return render_template_string(BASE_HTML, content=WELCOME_PAGE)
        role = session['role']
        if role == 'admin':
            return render_template_string(BASE_HTML, content=ADMIN_PANEL_HTML)
        elif role == 'client':
            return render_template_string(BASE_HTML, content=CLIENT_PANEL_HTML)
        else:
            return render_template_string(BASE_HTML, content=WORKER_PANEL_HTML)

    @app.route('/login', methods=['GET', 'POST'])
    def login_page():
        if request.method == 'POST':
            user = db_query("SELECT * FROM users WHERE username=? AND password=?",
                            (request.form['username'], hashlib.md5(request.form['password'].encode()).hexdigest()),
                            fetch_one=True)
            if user:
                session['user_id'] = user['id']
                session['username'] = user['username']
                session['full_name'] = user['full_name']
                session['role'] = user['role']
                flash(f"Добро пожаловать, {user['full_name']}!")
                return redirect(url_for('index'))
            flash("Неверный логин или пароль")
        return render_template_string(BASE_HTML, content=LOGIN_PAGE)

    @app.route('/register', methods=['GET', 'POST'])
    def register_page():
        if request.method == 'POST':
            full_name = request.form['full_name']
            username = request.form['username']
            password = hashlib.md5(request.form['password'].encode()).hexdigest()
            role = request.form['role']
            existing = db_query("SELECT id FROM users WHERE username=?", (username,), fetch_one=True)
            if existing:
                flash("Логин занят")
            else:
                db_query("INSERT INTO users (username, password, full_name, role) VALUES (?,?,?,?)",
                         (username, password, full_name, role), commit=True)
                flash("Регистрация успешна! Войдите.")
                return redirect(url_for('login_page'))
        return render_template_string(BASE_HTML, content=REGISTER_PAGE)

    @app.route('/logout')
    def logout():
        session.clear()
        flash("Вы вышли")
        return redirect(url_for('index'))

    @app.route('/admin_report_print')
    def admin_report_print():
        if not session.get('user_id') or session['role'] != 'admin':
            return redirect(url_for('login_page'))
        rows = db_query(
            'SELECT r.*, u.full_name as worker_name FROM requests r LEFT JOIN users u ON r.assigned_worker_id = u.id ORDER BY r.created_at DESC',
            fetch_all=True)
        html = f'''<!DOCTYPE html><html><head><meta charset="UTF-8"><title>Отчёт</title><style>body{{font-family:Arial;margin:20px}} table{{border-collapse:collapse;width:100%}} th,td{{border:1px solid #ccc;padding:8px}} th{{background:#f0f0f0}}</style></head><body><h1>Отчёт по заявкам</h1><p>Дата: {datetime.now().strftime('%d.%m.%Y %H:%M')}</p><table><thead><tr><th>ID</th><th>Клиент</th><th>Телефон</th><th>Услуга</th><th>Статус</th><th>Исполнитель</th><th>Дата создания</th></tr></thead><tbody>'''
        for r in rows:
            html += f"<tr><td class=\"p-2\">{r['id']}</td><td class=\"p-2\">{r['customer_name']}</td><td class=\"p-2\">{r['phone']}</td><td class=\"p-2\">{r['service_type']}</td><td class=\"p-2\">{r['status']}</td><td class=\"p-2\">{r['worker_name'] or ''}</td><td class=\"p-2\">{r['created_at']}</td></tr>"
        html += '</tbody></table><p>Сохраните как PDF через печать браузера.</p></body></html>'
        return html

    # ------------------- API -------------------
    @app.route('/api/workers')
    def api_workers():
        rows = db_query("SELECT id, full_name FROM users WHERE role='worker'", fetch_all=True)
        return jsonify([dict(r) for r in rows])

    @app.route('/api/resources')
    def api_resources():
        if not session.get('user_id') or session['role'] != 'admin':
            return jsonify({'error': 'Доступ запрещён'}), 403
        rows = db_query("SELECT * FROM resources", fetch_all=True)
        return jsonify([dict(r) for r in rows])

    @app.route('/api/resource', methods=['POST'])
    def add_resource():
        if not session.get('user_id') or session['role'] != 'admin':
            return jsonify({'error': 'Доступ запрещён'}), 403
        data = request.json
        db_query("INSERT INTO resources (name, type, quantity) VALUES (?,?,?)",
                 (data['name'], data['type'], data['quantity']), commit=True)
        return jsonify({'ok': True})

    @app.route('/api/resource/<int:rid>', methods=['PUT'])
    def update_resource(rid):
        if not session.get('user_id') or session['role'] != 'admin':
            return jsonify({'error': 'Доступ запрещён'}), 403
        db_query("UPDATE resources SET quantity=? WHERE id=?", (request.json['quantity'], rid), commit=True)
        return jsonify({'ok': True})

    @app.route('/api/resource/<int:rid>', methods=['DELETE'])
    def del_resource(rid):
        if not session.get('user_id') or session['role'] != 'admin':
            return jsonify({'error': 'Доступ запрещён'}), 403
        db_query("DELETE FROM resources WHERE id=?", (rid,), commit=True)
        return jsonify({'ok': True})

    @app.route('/api/seasons')
    def api_seasons():
        if not session.get('user_id') or session['role'] != 'admin':
            return jsonify({'error': 'Доступ запрещён'}), 403
        rows = db_query("SELECT * FROM seasons", fetch_all=True)
        return jsonify([dict(r) for r in rows])

    @app.route('/api/season', methods=['POST'])
    def add_season():
        if not session.get('user_id') or session['role'] != 'admin':
            return jsonify({'error': 'Доступ запрещён'}), 403
        data = request.json
        db_query("INSERT INTO seasons (name, period, recommendations) VALUES (?,?,?)",
                 (data['name'], data['period'], data['recommendations']), commit=True)
        return jsonify({'ok': True})

    @app.route('/api/season/<int:sid>', methods=['PUT'])
    def update_season(sid):
        if not session.get('user_id') or session['role'] != 'admin':
            return jsonify({'error': 'Доступ запрещён'}), 403
        db_query("UPDATE seasons SET recommendations=? WHERE id=?", (request.json['recommendations'], sid), commit=True)
        return jsonify({'ok': True})

    @app.route('/api/season/<int:sid>', methods=['DELETE'])
    def del_season(sid):
        if not session.get('user_id') or session['role'] != 'admin':
            return jsonify({'error': 'Доступ запрещён'}), 403
        db_query("DELETE FROM seasons WHERE id=?", (sid,), commit=True)
        return jsonify({'ok': True})

    @app.route('/api/stats')
    def stats():
        if not session.get('user_id') or session['role'] != 'admin':
            return jsonify({'error': 'Доступ запрещён'}), 403
        return jsonify({
            'total': db_query("SELECT COUNT(*) FROM requests", fetch_one=True)[0],
            'new': db_query("SELECT COUNT(*) FROM requests WHERE status='новая'", fetch_one=True)[0],
            'work': db_query("SELECT COUNT(*) FROM requests WHERE status='в работе'", fetch_one=True)[0],
            'done': db_query("SELECT COUNT(*) FROM requests WHERE status='выполнено'", fetch_one=True)[0],
            'cancelled': db_query("SELECT COUNT(*) FROM requests WHERE status='отменено'", fetch_one=True)[0]
        })

    @app.route('/api/requests')
    def admin_requests():
        if not session.get('user_id') or session['role'] != 'admin':
            return jsonify({'error': 'Доступ запрещён'}), 403
        search = request.args.get('search', '')
        status = request.args.get('status', 'все')
        worker_name = request.args.get('worker', 'все')
        query = 'SELECT r.*, u.full_name as worker_name, u.id as worker_id FROM requests r LEFT JOIN users u ON r.assigned_worker_id = u.id WHERE 1=1'
        params = []
        if search:
            query += " AND r.customer_name LIKE ?"; params.append(f"%{search}%")
        if status != 'все':
            query += " AND r.status = ?"; params.append(status)
        if worker_name != 'все':
            query += " AND u.full_name = ?"; params.append(worker_name)
        query += " ORDER BY r.created_at DESC"
        rows = db_query(query, params, fetch_all=True)
        today_count = db_query("SELECT COUNT(*) FROM requests WHERE DATE(created_at)=DATE('now')", fetch_one=True)[0]
        workers = db_query("SELECT id, full_name FROM users WHERE role='worker'", fetch_all=True)
        workers_list = [{'id': w['id'], 'full_name': w['full_name']} for w in workers]
        result = []
        for r in rows:
            d = dict(r)
            d['workers_list'] = workers_list
            result.append(d)
        return jsonify({'requests': result, 'today_count': today_count})

    @app.route('/api/client_requests')
    def client_requests():
        if not session.get('user_id') or session['role'] != 'client':
            return jsonify({'error': 'Доступ запрещён'}), 403
        name = request.args.get('name', '')
        if not name:
            return jsonify({'error': 'no name'}), 400
        rows = db_query("SELECT * FROM requests WHERE customer_name=? ORDER BY created_at DESC", (name,),
                        fetch_all=True)
        return jsonify({'requests': [dict(r) for r in rows]})

    @app.route('/api/worker_tasks')
    def worker_tasks():
        if not session.get('user_id') or session['role'] != 'worker':
            return jsonify({'error': 'Доступ запрещён'}), 403
        worker_id = request.args.get('worker_id', '')
        if not worker_id:
            return jsonify({'error': 'no worker_id'}), 400
        rows = db_query(
            'SELECT r.*, u.full_name as worker_name FROM requests r LEFT JOIN users u ON r.assigned_worker_id = u.id WHERE r.assigned_worker_id = ? ORDER BY r.created_at DESC',
            (worker_id,), fetch_all=True)
        return jsonify({'tasks': [dict(r) for r in rows]})

    @app.route('/api/request', methods=['POST'])
    def create_request():
        if not session.get('user_id') or session['role'] not in ('client', 'admin'):
            return jsonify({'error': 'Не авторизован'}), 401
        data = request.json
        if not re.search(r'[\d\+]{5,}', data.get('phone', '')):
            return jsonify({'status': 'error', 'message': 'Неверный телефон'}), 400
        db_query('INSERT INTO requests (customer_name, phone, address, description, service_type) VALUES (?,?,?,?,?)',
                 (data['name'], data['phone'], data['address'], data['description'], data['service_type']), commit=True)
        return jsonify({'status': 'ok', 'message': 'Заявка создана'})

    @app.route('/api/request/<int:req_id>', methods=['PUT'])
    def update_request(req_id):
        if not session.get('user_id') or session['role'] != 'admin':
            return jsonify({'error': 'Доступ запрещён'}), 403
        data = request.json
        if 'assigned_worker_id' in data:
            worker_id = data['assigned_worker_id'] if data['assigned_worker_id'] else None
            db_query("UPDATE requests SET assigned_worker_id=? WHERE id=?", (worker_id, req_id), commit=True)
        elif 'status' in data:
            st = data['status']
            if st == 'в работе':
                db_query("UPDATE requests SET status=?, started_at=CURRENT_TIMESTAMP WHERE id=?", (st, req_id),
                         commit=True)
            elif st == 'выполнено':
                db_query("UPDATE requests SET status=?, completed_at=CURRENT_TIMESTAMP WHERE id=?", (st, req_id),
                         commit=True)
            else:
                db_query("UPDATE requests SET status=? WHERE id=?", (st, req_id), commit=True)
        else:
            db_query(
                'UPDATE requests SET customer_name=?, phone=?, address=?, service_type=?, description=? WHERE id=?',
                (data.get('customer_name'), data.get('phone'), data.get('address'), data.get('service_type'),
                 data.get('description'), req_id), commit=True)
        return jsonify({'status': 'ok'})

    @app.route('/api/request/<int:req_id>', methods=['DELETE'])
    def delete_request(req_id):
        if not session.get('user_id') or session['role'] != 'admin':
            return jsonify({'error': 'Доступ запрещён'}), 403
        db_query("DELETE FROM requests WHERE id=?", (req_id,), commit=True)
        return jsonify({'status': 'ok'})

    @app.route('/api/report/csv')
    def export_csv():
        if not session.get('user_id') or session['role'] != 'admin':
            return jsonify({'error': 'Доступ запрещён'}), 403
        rows = db_query(
            'SELECT r.id, r.customer_name, r.phone, r.address, r.description, r.service_type, r.status, u.full_name as worker_name, r.created_at, r.started_at, r.completed_at FROM requests r LEFT JOIN users u ON r.assigned_worker_id = u.id ORDER BY r.created_at DESC',
            fetch_all=True)
        output = io.StringIO()
        writer = csv.writer(output, delimiter=';')
        writer.writerow(
            ['ID', 'Клиент', 'Телефон', 'Адрес', 'Описание', 'Услуга', 'Статус', 'Исполнитель', 'Создана', 'Начата',
             'Завершена'])
        for r in rows:
            writer.writerow([r['id'], r['customer_name'], r['phone'], r['address'], r['description'], r['service_type'],
                             r['status'], r['worker_name'] or '', r['created_at'], r['started_at'], r['completed_at']])
        output.seek(0)
        return send_file(io.BytesIO(output.getvalue().encode('cp1251')), mimetype='text/csv', as_attachment=True,
                         download_name='report.csv')

    if __name__ == '__main__':
        if not os.environ.get('WERKZEUG_RUN_MAIN'):
            threading.Timer(1.5, lambda: webbrowser.open('http://127.0.0.1:5000')).start()
        app.run(debug=True, host='127.0.0.1', port=5000)

except Exception as e:
    print("Ошибка при запуске приложения:")
    traceback.print_exc()
    input("Нажмите Enter для выхода...")
    sys.exit(1)
